## vpp主程序

vpp主程序main函数位于`vpp/vnet/main.c`文件中。

### main函数

```
int
main (int argc, char *argv[])
{
  int i;
  vlib_main_t *vm = &vlib_global_main;
  void vl_msg_api_set_first_available_msg_id (u16);
  uword main_heap_size = (1ULL << 30);
  u8 *sizep;
  u32 size;
  int main_core = 1;
  cpu_set_t cpuset;
  void *main_heap;

#if __x86_64__
  CLIB_UNUSED (const char *msg)
    = "ERROR: This binary requires CPU with %s extensions.\n";
#define _(a,b)                                  \
    if (!clib_cpu_supports_ ## a ())            \
      {                                         \
	fprintf(stderr, msg, b);                \
	exit(1);                                \
      }

#if __AVX2__
  _(avx2, "AVX2")
#endif
#if __AVX__
    _(avx, "AVX")
#endif
#if __SSE4_2__
    _(sse42, "SSE4.2")
#endif
#if __SSE4_1__
    _(sse41, "SSE4.1")
#endif
#if __SSSE3__
    _(ssse3, "SSSE3")
#endif
#if __SSE3__
    _(sse3, "SSE3")
#endif
#undef _
#endif
    /*
     * Load startup config from file.
     * usage: vpp -c /etc/vpp/startup.conf
     */
    if ((argc == 3) && !strncmp (argv[1], "-c", 2))
    {
      FILE *fp;
      char inbuf[4096];
      int argc_ = 1;
      char **argv_ = NULL;
      char *arg = NULL;
      char *p;

      fp = fopen (argv[2], "r");
      if (fp == NULL)
	{
	  fprintf (stderr, "open configuration file '%s' failed\n", argv[2]);
	  return 1;
	}
      argv_ = calloc (1, sizeof (char *));
      if (argv_ == NULL)
	{
	  fclose (fp);
	  return 1;
	}
      arg = strndup (argv[0], 1024);
      if (arg == NULL)
	{
	  fclose (fp);
	  free (argv_);
	  return 1;
	}
      argv_[0] = arg;

      while (1)
	{
	  if (fgets (inbuf, 4096, fp) == 0)
	    break;
	  p = strtok (inbuf, " \t\n");
	  while (p != NULL)
	    {
	      if (*p == '#')
		break;
	      argc_++;
	      char **tmp = realloc (argv_, argc_ * sizeof (char *));
	      if (tmp == NULL)
		return 1;
	      argv_ = tmp;
	      arg = strndup (p, 1024);
	      if (arg == NULL)
		return 1;
	      argv_[argc_ - 1] = arg;
	      p = strtok (NULL, " \t\n");
	    }
	}

      fclose (fp);

      char **tmp = realloc (argv_, (argc_ + 1) * sizeof (char *));
      if (tmp == NULL)
	return 1;
      argv_ = tmp;
      argv_[argc_] = NULL;

      argc = argc_;
      argv = argv_;
    }

  /*
   * Look for and parse the "heapsize" config parameter.
   * Manual since none of the clib infra has been bootstrapped yet.
   *
   * Format: heapsize <nn>[mM][gG]
   */

  for (i = 1; i < (argc - 1); i++)
    {
      if (!strncmp (argv[i], "plugin_path", 11))
	{
	  if (i < (argc - 1))
	    vlib_plugin_path = argv[++i];
	}
      if (!strncmp (argv[i], "test_plugin_path", 16))
	{
	  if (i < (argc - 1))
	    vat_plugin_path = argv[++i];
	}
      else if (!strncmp (argv[i], "heapsize", 8))
	{
	  sizep = (u8 *) argv[i + 1];
	  size = 0;
	  while (*sizep >= '0' && *sizep <= '9')
	    {
	      size *= 10;
	      size += *sizep++ - '0';
	    }
	  if (size == 0)
	    {
	      fprintf
		(stderr,
		 "warning: heapsize parse error '%s', use default %lld\n",
		 argv[i], (long long int) main_heap_size);
	      goto defaulted;
	    }

	  main_heap_size = size;

	  if (*sizep == 'g' || *sizep == 'G')
	    main_heap_size <<= 30;
	  else if (*sizep == 'm' || *sizep == 'M')
	    main_heap_size <<= 20;
	}
      else if (!strncmp (argv[i], "main-core", 9))
	{
	  if (i < (argc - 1))
	    {
	      errno = 0;
	      unsigned long x = strtol (argv[++i], 0, 0);
	      if (errno == 0)
		main_core = x;
	    }
	}
    }

defaulted:

  /* set process affinity for main thread */
  CPU_ZERO (&cpuset);
  CPU_SET (main_core, &cpuset);
  pthread_setaffinity_np (pthread_self (), sizeof (cpu_set_t), &cpuset);

  /* Set up the plugin message ID allocator right now... */
  vl_msg_api_set_first_available_msg_id (VL_MSG_FIRST_AVAILABLE);

  /* Allocate main heap */
  if ((main_heap = clib_mem_init_thread_safe (0, main_heap_size)))
    {
      vlib_worker_thread_t tmp;

      /* Figure out which numa runs the main thread */
      vlib_get_thread_core_numa (&tmp, main_core);
      __os_numa_index = tmp.numa_id;

      /* and use the main heap as that numa's numa heap */
      clib_mem_set_per_numa_heap (main_heap);

      vm->init_functions_called = hash_create (0, /* value bytes */ 0);
      vpe_main_init (vm);
      return vlib_unix_main (argc, argv);
    }
  else
    {
      {
	int rv __attribute__ ((unused)) =
	  write (2, "Main heap allocation failure!\r\n", 31);
      }
      return 1;
    }
}
```