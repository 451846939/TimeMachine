# top内存读取与cgroup读取内存

本文内核源码基于linux 6.5



## top



首先我们看一下top是如何读取内存的

在procps源码中可以得知top中进程信息由`task_show` 进行数据转换

```c
static const char *task_show (const WIN_t *q, const proc_t *p) {
  .......
 #define pages2K(n)  (unsigned long)( (n) << Pg2K_shft )
   static char rbuf[ROWMINSIZ];
   char *rp;
   int x;

   // we must begin a row with a possible window number in mind...
   *(rp = rbuf) = '\0';
   if (Rc.mode_altscr) rp = scat(rp, " ");

   for (x = 0; x < q->maxpflgs; x++) {
      const char *cp;
      FLG_t       i = q->procflgs[x];
      #define S   Fieldstab[i].scale        // these used to be variables
      #define W   Fieldstab[i].width        // but it's much better if we
      #define Js  CHKw(q, Show_JRSTRS)      // represent them as #defines
      #define Jn  CHKw(q, Show_JRNUMS)      // and only exec code if used

      switch (i) {
					.........
         case EU_CGR:
            makeVAR(p->cgroup[0]);
            break;
         case EU_CPN:
            cp = make_num(p->processor, W, Jn, AUTOX_NO);
            break;
         case EU_CPU:
         {  float u = (float)p->pcpu * Frame_etscale;
            /* process can't use more %cpu than number of threads it has
             ( thanks Jaromir Capik <jcapik@redhat.com> ) */
            if (u > 100.0 * p->nlwp) u = 100.0 * p->nlwp;
            if (u > Cpu_pmax) u = Cpu_pmax;
            cp = scale_pcnt(u, W, Jn);
         }
            break;
         case EU_GRP:
            cp = make_str(p->egroup, W, Js, EU_GRP);
            break;
         case EU_MEM:
            cp = scale_pcnt((float)pages2K(p->resident) * 100 / kb_main_total, W, Jn);
            break;
         case EU_RES:
            cp = scale_mem(S, pages2K(p->resident), W, Jn);
            break;
         case EU_SHR:
            cp = scale_mem(S, pages2K(p->share), W, Jn);
            break;
         case EU_SWP:
            cp = scale_mem(S, p->vm_swap, W, Jn);
            break;
         case EU_USE:
            cp = scale_mem(S, (p->vm_swap + pages2K(p->resident)), W, Jn);
            break;
         case EU_VRT:
            cp = scale_mem(S, pages2K(p->size), W, Jn);
            break;
         case EU_WCH:
         {  const char *u;
            if (No_ksyms)
               u = hex_make(p->wchan, 0);
            else
               u = lookup_wchan(p->wchan, p->tid);
            cp = make_str(u, W, Js, EU_WCH);
         }
            break;
         default:                 // keep gcc happy
            continue;
....
      } // end: switch 'procflag'
....
   } // end: for 'maxpflgs'
	.......
} // end: task_show

static const char *scale_mem (int target, unsigned long num, int width, int justr) {
#ifndef NOBOOST_MEMS
   //                               SK_Kb   SK_Mb      SK_Gb      SK_Tb      SK_Pb      SK_Eb
   static const char *fmttab[] =  { "%.0f", "%#.1f%c", "%#.3f%c", "%#.3f%c", "%#.3f%c", NULL };
#else
   static const char *fmttab[] =  { "%.0f", "%.0f%c",  "%.0f%c",  "%.0f%c",  "%.0f%c",  NULL };
#endif
   static char buf[SMLBUFSIZ];
   float scaled_num;
   char *psfx;
   int i;

   buf[0] = '\0';
   if (Rc.zero_suppress && 0 >= num)
      goto end_justifies;

   scaled_num = num;
   for (i = SK_Kb, psfx = Scaled_sfxtab; i < SK_Eb; psfx++, i++) {
      if (i >= target
      && (width >= snprintf(buf, sizeof(buf), fmttab[i], scaled_num, *psfx)))
         goto end_justifies;
      scaled_num /= 1024.0;
   }

   // well shoot, this outta' fit...
   snprintf(buf, sizeof(buf), "?");
end_justifies:
   return justify_pad(buf, width, justr);
} // end: scale_mem
```

可以看到`RES`读取的是`/proc/{pid}/statm` 中的`resident`，`SHR`读取的是`share`之后计算公式如果转换为MB

`({count} << Pg2K_shft) / 1024` 其中`Pg2K_shft` 计算公式为

```c
//一般为4096	可以运行getconf PAGE_SIZE查看
page_bytes = sysconf(_SC_PAGESIZE)   
// get virtual page stuff
i = page_bytes; // from sysinfo.c, at lib init
while(i > 1024) { i >>= 1; Pg2K_shft++; }
```

在内核中该文件的实现如下

```c
unsigned long task_statm(struct mm_struct *mm,
			 unsigned long *shared, unsigned long *text,
			 unsigned long *data, unsigned long *resident)
{
	*shared = get_mm_counter(mm, MM_FILEPAGES) +
			get_mm_counter(mm, MM_SHMEMPAGES);
	*text = (PAGE_ALIGN(mm->end_code) - (mm->start_code & PAGE_MASK))
								>> PAGE_SHIFT;
	*data = mm->data_vm + mm->stack_vm;
	*resident = *shared + get_mm_counter(mm, MM_ANONPAGES);
	return mm->total_vm;
}

/*
 * When updating this, please also update struct resident_page_types[] in
 * kernel/fork.c
 */
enum {
	MM_FILEPAGES,	/* Resident file mapping pages */
	MM_ANONPAGES,	/* Resident anonymous pages */
	MM_SWAPENTS,	/* Anonymous swap entries */
	MM_SHMEMPAGES,	/* Resident shared memory pages */
	NR_MM_COUNTERS
};
```

可以看到计算如下

**SHR->shared=MM_FILEPAGES+MM_SHMEMPAGES **

**RES->resident=shared+MM_ANONPAGES**



`get_mm_counter`是如何读取的呢，函数定义如下主要读取的是`mm->rss_stat[member]`

```c
static inline unsigned long get_mm_counter(struct mm_struct *mm, int member)
{
	return percpu_counter_read_positive(&mm->rss_stat[member]);
}
```





什么时候进行的计数呢流程如下:

```c
do_page_fault
      ->handle_mm_fault
      ->__handle_mm_fault
      ->handle_pte_fault
      ->do_pte_missing
      ->do_anonymous_page
      ->inc_mm_counter
  
static inline void inc_mm_counter(struct mm_struct *mm, int member)
{
	percpu_counter_inc(&mm->rss_stat[member]);

	mm_trace_rss_stat(mm, member);
}
```



至此我们也可以看到这种主要针对的是进程来进行统计计算





## cgroup



对于stat文件背后调用的方法

```c
memory_stat_show
  ->memory_stat_format
  ->memcg_stat_format
static struct cftype memory_files[] = {	
  ....
	{
		.name = "stat",
		.seq_show = memory_stat_show,
	},
  .....
}
static void memcg_stat_format(struct mem_cgroup *memcg, struct seq_buf *s)
{
	int i;

	/*
	 * Provide statistics on the state of the memory subsystem as
	 * well as cumulative event counters that show past behavior.
	 *
	 * This list is ordered following a combination of these gradients:
	 * 1) generic big picture -> specifics and details
	 * 2) reflecting userspace activity -> reflecting kernel heuristics
	 *
	 * Current memory state:
	 */
	mem_cgroup_flush_stats();

	for (i = 0; i < ARRAY_SIZE(memory_stats); i++) {
		u64 size;

		size = memcg_page_state_output(memcg, memory_stats[i].idx);
		seq_buf_printf(s, "%s %llu\n", memory_stats[i].name, size);

		if (unlikely(memory_stats[i].idx == NR_SLAB_UNRECLAIMABLE_B)) {
			size += memcg_page_state_output(memcg,
							NR_SLAB_RECLAIMABLE_B);
			seq_buf_printf(s, "slab %llu\n", size);
		}
	}

	/* Accumulated memory events */
	seq_buf_printf(s, "pgscan %lu\n",
		       memcg_events(memcg, PGSCAN_KSWAPD) +
		       memcg_events(memcg, PGSCAN_DIRECT) +
		       memcg_events(memcg, PGSCAN_KHUGEPAGED));
	seq_buf_printf(s, "pgsteal %lu\n",
		       memcg_events(memcg, PGSTEAL_KSWAPD) +
		       memcg_events(memcg, PGSTEAL_DIRECT) +
		       memcg_events(memcg, PGSTEAL_KHUGEPAGED));

	for (i = 0; i < ARRAY_SIZE(memcg_vm_event_stat); i++) {
		if (memcg_vm_event_stat[i] == PGPGIN ||
		    memcg_vm_event_stat[i] == PGPGOUT)
			continue;

		seq_buf_printf(s, "%s %lu\n",
			       vm_event_name(memcg_vm_event_stat[i]),
			       memcg_events(memcg, memcg_vm_event_stat[i]));
	}

	/* The above should easily fit into one page */
	WARN_ON_ONCE(seq_buf_has_overflowed(s));
}
/*计算展示数值*/
static inline unsigned long memcg_page_state_output(struct mem_cgroup *memcg,
						    int item)
{
	return memcg_page_state(memcg, item) * memcg_page_state_unit(item);
}

unsigned long memcg_page_state(struct mem_cgroup *memcg, int idx)
{
	long x = READ_ONCE(memcg->vmstats->state[idx]);
#ifdef CONFIG_SMP
	if (x < 0)
		x = 0;
#endif
	return x;
}

/* Translate stat items to the correct unit for memory.stat output */
static int memcg_page_state_unit(int item)
{
	switch (item) {
	case MEMCG_PERCPU_B:
	case MEMCG_ZSWAP_B:
	case NR_SLAB_RECLAIMABLE_B:
	case NR_SLAB_UNRECLAIMABLE_B:
	case WORKINGSET_REFAULT_ANON:
	case WORKINGSET_REFAULT_FILE:
	case WORKINGSET_ACTIVATE_ANON:
	case WORKINGSET_ACTIVATE_FILE:
	case WORKINGSET_RESTORE_ANON:
	case WORKINGSET_RESTORE_FILE:
	case WORKINGSET_NODERECLAIM:
		return 1;
	case NR_KERNEL_STACK_KB:
		return SZ_1K;
	default:
		return PAGE_SIZE;
	}
}
```



关键代码为`memcg_page_state_output * memcg_page_state_unit`如果`PAGE_SIZE`和上述一样都是4096那么换算mb的公式为`count*4096/1024/1024` ,`memcg_page_state_output`进行了`memcg->vmstats->state[idx]`的读取，那么我们是如何写入的呢



再看一下`memory_cgrp_subsys` cgroup子系统

```c
struct cgroup_subsys memory_cgrp_subsys = {
	.css_alloc = mem_cgroup_css_alloc,
	.css_online = mem_cgroup_css_online,
	.css_offline = mem_cgroup_css_offline,
	.css_released = mem_cgroup_css_released,
	.css_free = mem_cgroup_css_free,
	.css_reset = mem_cgroup_css_reset,
	.css_rstat_flush = mem_cgroup_css_rstat_flush,
	.can_attach = mem_cgroup_can_attach,
	.attach = mem_cgroup_attach,
	.cancel_attach = mem_cgroup_cancel_attach,
	.post_attach = mem_cgroup_move_task,
	.dfl_cftypes = memory_files,
	.legacy_cftypes = mem_cgroup_legacy_files,
	.early_init = 0,
};
```



其中`css_rstat_flush`对应`mem_cgroup_css_rstat_flush`可以看到在`memcg->vmstats->state`更新数据



```c
static void mem_cgroup_css_rstat_flush(struct cgroup_subsys_state *css, int cpu)
{
	struct mem_cgroup *memcg = mem_cgroup_from_css(css);
	struct mem_cgroup *parent = parent_mem_cgroup(memcg);
	struct memcg_vmstats_percpu *statc;
	long delta, v;
	int i, nid;

	statc = per_cpu_ptr(memcg->vmstats_percpu, cpu);

	for (i = 0; i < MEMCG_NR_STAT; i++) {
		/*
		 * Collect the aggregated propagation counts of groups
		 * below us. We're in a per-cpu loop here and this is
		 * a global counter, so the first cycle will get them.
		 */
		delta = memcg->vmstats->state_pending[i];
		if (delta)
			memcg->vmstats->state_pending[i] = 0;

		/* Add CPU changes on this level since the last flush */
		v = READ_ONCE(statc->state[i]);
		if (v != statc->state_prev[i]) {
			delta += v - statc->state_prev[i];
			statc->state_prev[i] = v;
		}

		if (!delta)
			continue;

		/* Aggregate counts on this level and propagate upwards */
		memcg->vmstats->state[i] += delta;
		if (parent)
			parent->vmstats->state_pending[i] += delta;
	}

	for (i = 0; i < NR_MEMCG_EVENTS; i++) {
		delta = memcg->vmstats->events_pending[i];
		if (delta)
			memcg->vmstats->events_pending[i] = 0;

		v = READ_ONCE(statc->events[i]);
		if (v != statc->events_prev[i]) {
			delta += v - statc->events_prev[i];
			statc->events_prev[i] = v;
		}

		if (!delta)
			continue;

		memcg->vmstats->events[i] += delta;
		if (parent)
			parent->vmstats->events_pending[i] += delta;
	}

	for_each_node_state(nid, N_MEMORY) {
		struct mem_cgroup_per_node *pn = memcg->nodeinfo[nid];
		struct mem_cgroup_per_node *ppn = NULL;
		struct lruvec_stats_percpu *lstatc;

		if (parent)
			ppn = parent->nodeinfo[nid];

		lstatc = per_cpu_ptr(pn->lruvec_stats_percpu, cpu);

		for (i = 0; i < NR_VM_NODE_STAT_ITEMS; i++) {
			delta = pn->lruvec_stats.state_pending[i];
			if (delta)
				pn->lruvec_stats.state_pending[i] = 0;

			v = READ_ONCE(lstatc->state[i]);
			if (v != lstatc->state_prev[i]) {
				delta += v - lstatc->state_prev[i];
				lstatc->state_prev[i] = v;
			}

			if (!delta)
				continue;

			pn->lruvec_stats.state[i] += delta;
			if (ppn)
				ppn->lruvec_stats.state_pending[i] += delta;
		}
	}
}

unsigned long memcg_page_state(struct mem_cgroup *memcg, int idx)
{
	long x = READ_ONCE(memcg->vmstats->state[idx]);
#ifdef CONFIG_SMP
	if (x < 0)
		x = 0;
#endif
	return x;
}
```



`per_cpu_ptr(memcg->vmstats_percpu, cpu)`会读取我们的`vmstats_percpu`那这里又是何时修改呢

在`__mod_lruvec_state`中可以看到

```c
/* Update memcg */
__this_cpu_add(memcg->vmstats_percpu->state[idx], valDi);
/* Update lruvec */
__this_cpu_add(pn->lruvec_stats_percpu->state[idx],val);
```

可以追踪到`__mod_memcg_state`、`__mod_memcg_lruvec_state`

```text
__vmalloc_node_range
      ->__vmalloc_area_node
      ->mod_memcg_page_state
      ->mod_memcg_state
      ->__mod_memcg_state
      ->__this_cpu_add(memcg->vmstats_percpu->state[idx], val);
```

当然还有被动的`do_page_fault`

```c
do_page_fault
      ->handle_mm_fault
      ->__handle_mm_fault
      ->handle_pte_fault
      ->do_pte_missing
      ->do_anonymous_page
      ->mem_cgroup_charge
      ->__mem_cgroup_charge
      ->charge_memcg
      ->mem_cgroup_charge_statistics

static void mem_cgroup_charge_statistics(struct mem_cgroup *memcg,
					 int nr_pages)
{
	/* pagein of a big page is an event. So, ignore page size */
	if (nr_pages > 0)
		__count_memcg_events(memcg, PGPGIN, 1);
	else {
		__count_memcg_events(memcg, PGPGOUT, 1);
		nr_pages = -nr_pages; /* for event */
	}

	__this_cpu_add(memcg->vmstats_percpu->nr_page_events, nr_pages);
}
```

以上就是增加cgroup内存计数的流程





