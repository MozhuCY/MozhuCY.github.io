title: io_file学习
categories: 
- PWN
---
# io_file学习

## 一些总结

### house of orange

- 首先从house of orange说起，这一般是接触io_file最早的时候，一般hoo的题目中都是没有free的，所以只能一头malloc下去
- 这个时候只需要一个堆溢出就够了，覆盖topchunk的size，简单都是只留第24bit位，剩余高位变成0，当然这样也直接绕过了关于top_chunk的一些检测，如内存页对齐，即加起来必须是0x1000的整数倍，size大于MINSIZE，prev_inuse必须为1
- 绕过这些检测以后，因为topchunk的size小于所申请的size，就可以触发sysmalloc中的_init_free函数,得到一个unsorted bin的堆块
- 接下来就是io_file的部分，再次申请堆块，切割这个unsorted堆块，剩下一个size为0x60的堆块，此时，溢出这个堆块，进行unsorted bin attack，具体构造如下
- 首先size不能变，依旧是0x61，然后在bk的位置写入_IO_list_all - 0x10，这两点的目的是在下一次malloc一个大于剩余堆块大小的堆块时，将此堆块放入smallbin中因为size为0x61，故此时这个chunk会被放到相对main_arena偏移为0x68的地方，而这个时候_IO_list_all被我们写入了main_arena偏移为88的值，也就是&unsorted bin的地址，在_IO_FILE中，_IO_list_all作为文件流头指针，FILE结构体通过_chain域进行连接，注意iofile的特性，在文件结构体异常的时候，会根据_chain指针的偏移遍历到下一个file，所以通过unsorted bin attack劫持_IO_list_all后，根据偏移，也就是在0x60归入的smallbins的bins中的偏移
-

```c
struct malloc_state
{
  /* Serialize access.  */
  mutex_t mutex;

  /* Flags (formerly in max_fast).  */
  int flags;

  /* Fastbins */
  mfastbinptr fastbinsY[NFASTBINS];

  /* Base of the topmost chunk -- not otherwise kept in a bin */
  mchunkptr top;

  /* The remainder from the most recent split of a small request */
  mchunkptr last_remainder;

  /* Normal bins packed as described above */
  mchunkptr bins[NBINS * 2 - 2];

  /* Bitmap of bins */
  unsigned int binmap[BINMAPSIZE];

  /* Linked list */
  struct malloc_state *next;

  /* Linked list for free arenas.  Access to this field is serialized
     by free_list_lock in arena.c.  */
  struct malloc_state *next_free;

  /* Number of threads attached to this arena.  0 if the arena is on
     the free list.  Access to this field is serialized by
     free_list_lock in arena.c.  */
  INTERNAL_SIZE_T attached_threads;

  /* Memory allocated from the system in this arena.  */
  INTERNAL_SIZE_T system_mem;
  INTERNAL_SIZE_T max_system_mem;
};
```

- 下面是_IO_FILE的结构体

```c
struct _IO_FILE {
  int _flags;		/* High-order word is _IO_MAGIC; rest is flags. */
#define _IO_file_flags _flags

  /* The following pointers correspond to the C++ streambuf protocol. */
  /* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
  char* _IO_read_ptr;	/* Current read pointer */
  char* _IO_read_end;	/* End of get area. */
  char* _IO_read_base;	/* Start of putback+get area. */
  char* _IO_write_base;	/* Start of put area. */
  char* _IO_write_ptr;	/* Current put pointer. */
  char* _IO_write_end;	/* End of put area. */
  char* _IO_buf_base;	/* Start of reserve area. */
  char* _IO_buf_end;	/* End of reserve area. */
  /* The following fields are used to support backing up and undo. */
  char *_IO_save_base; /* Pointer to start of non-current get area. */
  char *_IO_backup_base;  /* Pointer to first valid character of backup area */
  char *_IO_save_end; /* Pointer to end of non-current get area. */

  struct _IO_marker *_markers;

  struct _IO_FILE *_chain;

  int _fileno;
#if 0
  int _blksize;
#else
  int _flags2;
#endif
  _IO_off_t _old_offset; /* This used to be _offset but it's too small.  */

#define __HAVE_COLUMN /* temporary */
  /* 1+column number of pbase(); 0 is unknown. */
  unsigned short _cur_column;
  signed char _vtable_offset;
  char _shortbuf[1];

  /*  char* _save_gptr;  char* _save_egptr; */

  _IO_lock_t *_lock;
#ifdef _IO_USE_OLD_IO_FILE
}
```

- 在劫持了_IO_list_all以后，我们之前剩下的0x60的堆块放入到了smallbin中，这时需要我们继续构造_IO_FILE_plus结构体，2.23中，只需要将相对偏移后的vtable指针指向我们构造好的位置，即可控制_IO_OVERFLOW函数指针，最后触发_IO_OVERFLOW时getshell
- 这个时候要构造FILE结构体成员的值，触发IOoverflow，向上回溯，在最后一次malloc的时候，因为之前的chunk破坏了unsorted bin的双向链表，所以在后一次malloc时，会触发异常，从而调用malloc_printerr函数

```c
static void
malloc_printerr (int action, const char *str, void *ptr, mstate ar_ptr)
{
  /* Avoid using this arena in future.  We do not attempt to synchronize this
     with anything else because we minimally want to ensure that __libc_message
     gets its resources safely without stumbling on the current corruption.  */
  if (ar_ptr)
    set_arena_corrupt (ar_ptr);

  if ((action & 5) == 5)
    __libc_message (action & 2, "%s\n", str);
  else if (action & 1)
    {
      char buf[2 * sizeof (uintptr_t) + 1];

      buf[sizeof (buf) - 1] = '\0';
      char *cp = _itoa_word ((uintptr_t) ptr, &buf[sizeof (buf) - 1], 16, 0);
      while (cp > buf)
        *--cp = '0';

      __libc_message (action & 2, "*** Error in `%s': %s: 0x%s ***\n",
                      __libc_argv[0] ? : "<unknown>", str, cp);
    }
  else if (action & 2)
    abort ();
}
```

- 可以看到函数中调用了__libc_message函数

```c
void
__libc_message (int do_abort, const char *fmt, ...)
{
  va_list ap;
  int fd = -1;

  va_start (ap, fmt);

#ifdef FATAL_PREPARE
  FATAL_PREPARE;
#endif

  /* Open a descriptor for /dev/tty unless the user explicitly
     requests errors on standard error.  */
  const char *on_2 = __libc_secure_getenv ("LIBC_FATAL_STDERR_");
  if (on_2 == NULL || *on_2 == '\0')
    fd = open_not_cancel_2 (_PATH_TTY, O_RDWR | O_NOCTTY | O_NDELAY);

  if (fd == -1)
    fd = STDERR_FILENO;

  struct str_list *list = NULL;
  int nlist = 0;

  const char *cp = fmt;
  while (*cp != '\0')
    {
      /* Find the next "%s" or the end of the string.  */
      const char *next = cp;
      while (next[0] != '%' || next[1] != 's')
	{
	  next = __strchrnul (next + 1, '%');

	  if (next[0] == '\0')
	    break;
	}

      /* Determine what to print.  */
      const char *str;
      size_t len;
      if (cp[0] == '%' && cp[1] == 's')
	{
	  str = va_arg (ap, const char *);
	  len = strlen (str);
	  cp += 2;
	}
      else
	{
	  str = cp;
	  len = next - cp;
	  cp = next;
	}

      struct str_list *newp = alloca (sizeof (struct str_list));
      newp->str = str;
      newp->len = len;
      newp->next = list;
      list = newp;
      ++nlist;
    }

  bool written = false;
  if (nlist > 0)
    {
      struct iovec *iov = alloca (nlist * sizeof (struct iovec));
      ssize_t total = 0;

      for (int cnt = nlist - 1; cnt >= 0; --cnt)
	{
	  iov[cnt].iov_base = (char *) list->str;
	  iov[cnt].iov_len = list->len;
	  total += list->len;
	  list = list->next;
	}

      written = WRITEV_FOR_FATAL (fd, iov, nlist, total);

      if (do_abort)
	{
	  total = ((total + 1 + GLRO(dl_pagesize) - 1)
		   & ~(GLRO(dl_pagesize) - 1));
	  struct abort_msg_s *buf = __mmap (NULL, total,
					    PROT_READ | PROT_WRITE,
					    MAP_ANON | MAP_PRIVATE, -1, 0);
	  if (__glibc_likely (buf != MAP_FAILED))
	    {
	      buf->size = total;
	      char *wp = buf->msg;
	      for (int cnt = 0; cnt < nlist; ++cnt)
		wp = mempcpy (wp, iov[cnt].iov_base, iov[cnt].iov_len);
	      *wp = '\0';

	      /* We have to free the old buffer since the application might
		 catch the SIGABRT signal.  */
	      struct abort_msg_s *old = atomic_exchange_acq (&__abort_msg,
							     buf);
	      if (old != NULL)
		__munmap (old, old->size);
	    }
	}
    }

  va_end (ap);

  if (do_abort)
    {
      BEFORE_ABORT (do_abort, written, fd);

      /* Kill the application.  */
      abort ();
    }
}
```

- 这里又调用了abort函数，而abort函数的内部

```c
void abort (void)
{
  struct sigaction act;
  sigset_t sigs;

  /* First acquire the lock.  */
  __libc_lock_lock_recursive (lock);

  /* Now it's for sure we are alone.  But recursive calls are possible.  */

  /* Unlock SIGABRT.  */
  if (stage == 0)
    {
      ++stage;
      if (__sigemptyset (&sigs) == 0 &&
	  __sigaddset (&sigs, SIGABRT) == 0)
	__sigprocmask (SIG_UNBLOCK, &sigs, (sigset_t *) NULL);
    }

  /* Flush all streams.  We cannot close them now because the user
     might have registered a handler for SIGABRT.  */
  if (stage == 1)
    {
      ++stage;
      fflush (NULL);
    }

  /* Send signal which possibly calls a user handler.  */
  if (stage == 2)
    {
      /* This stage is special: we must allow repeated calls of
	 `abort' when a user defined handler for SIGABRT is installed.
	 This is risky since the `raise' implementation might also
	 fail but I don't see another possibility.  */
      int save_stage = stage;

      stage = 0;
      __libc_lock_unlock_recursive (lock);

      raise (SIGABRT);

      __libc_lock_lock_recursive (lock);
      stage = save_stage + 1;
    }

  /* There was a handler installed.  Now remove it.  */
  if (stage == 3)
    {
      ++stage;
      memset (&act, '\0', sizeof (struct sigaction));
      act.sa_handler = SIG_DFL;
      __sigfillset (&act.sa_mask);
      act.sa_flags = 0;
      __sigaction (SIGABRT, &act, NULL);
    }

  /* Now close the streams which also flushes the output the user
     defined handler might has produced.  */
  if (stage == 4)
    {
      ++stage;
      __fcloseall ();
    }

  /* Try again.  */
  if (stage == 5)
    {
      ++stage;
      raise (SIGABRT);
    }

  /* Now try to abort using the system specific command.  */
  if (stage == 6)
    {
      ++stage;
      ABORT_INSTRUCTION;
    }

  /* If we can't signal ourselves and the abort instruction failed, exit.  */
  if (stage == 7)
    {
      ++stage;
      _exit (127);
    }

  /* If even this fails try to use the provided instruction to crash
     or otherwise make sure we never return.  */
  while (1)
    /* Try for ever and ever.  */
    ABORT_INSTRUCTION;
}
```

- 继续追踪发现调用了fflush函数

```c
int
_IO_flush_all_lockp (int do_lock)
{
  int result = 0;
  struct _IO_FILE *fp;
  int last_stamp;

#ifdef _IO_MTSAFE_IO
  __libc_cleanup_region_start (do_lock, flush_cleanup, NULL);
  if (do_lock)
    _IO_lock_lock (list_all_lock);
#endif

  last_stamp = _IO_list_all_stamp;
  fp = (_IO_FILE *) _IO_list_all; // 用到了_IO_list_all
  while (fp != NULL) //此时fp为mainarena+88
    {
      run_fp = fp;
      if (do_lock)
	_IO_flockfile (fp);

      if (((fp->_mode <= 0 && fp->_IO_write_ptr > fp->_IO_write_base) //需要满足的条件在这里
#if defined _LIBC || defined _GLIBCPP_USE_WCHAR_T
	   || (_IO_vtable_offset (fp) == 0
	       && fp->_mode > 0 && (fp->_wide_data->_IO_write_ptr
				    > fp->_wide_data->_IO_write_base))
#endif
	   )
	  && _IO_OVERFLOW (fp, EOF) == EOF)
	result = EOF;

      if (do_lock)
	_IO_funlockfile (fp);
      run_fp = NULL;

      if (last_stamp != _IO_list_all_stamp)
	{
	  /* Something was added to the list.  Start all over again.  */
	  fp = (_IO_FILE *) _IO_list_all;
	  last_stamp = _IO_list_all_stamp;
	}
      else
	fp = fp->_chain;
    }

#ifdef _IO_MTSAFE_IO
  if (do_lock)
    _IO_lock_unlock (list_all_lock);
  __libc_cleanup_region_end (0);
#endif

  return result;
}
```

- 也就是在这个while循环中，我们完成了所有的操作，首先拿到了io list all指向的指针后，一直后向遍历直到为空，构造时要满足`fp->_mode <= 0 && fp->_IO_write_ptr > fp->_IO_write_base`
- 最后要调用_IO_OVERFLOW (fp, EOF)，所以/bin/sh字符串应该写道fp的位置,结合上面的unsorted bin attack的条件，需要构造

```
-----------------------
|      /bin/sh\x00    |
-----------------------
|         0x61        |
-----------------------
|         -1          |
-----------------------
|  &_IO_list_all-0x10 |
-----------------------
|          0          |
-----------------------
|          0          |
-----------------------
|          1          |
-----------------------
|                     |
-----------------------
|                     |
-----------------------
|                     |
-----------------------
|                     |
-----------------------
```