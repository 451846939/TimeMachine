# top内存读取与cgroup读取内存





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
```

可以看到`RES`读取的是`/proc/{pid}/statm` 中的`resident`，`SHR`读取的是`share`之后计算公式如果转换为MB

`({count} << Pg2K_shft) / 1024 / 1024` 其中`Pg2K_shft` 计算公式为

```c
//一般为4096	可以运行getconf PAGE_SIZE查看
page_bytes = sysconf(_SC_PAGESIZE)   
// get virtual page stuff
i = page_bytes; // from sysinfo.c, at lib init
while(i > 1024) { i >>= 1; Pg2K_shft++; }
```

在内核中该文件的实现

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

SHR->shared=MM_FILEPAGES+MM_SHMEMPAGES 

RES->resident=shared+MM_ANONPAGES



至此我们也可以看到这种主要针对的是进程来进行统计计算





## cgroup



先看一下memory_cgrp_subsys cgroup子系统

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



其中`css_rstat_flush`对应`mem_cgroup_css_rstat_flush`可以看到在memcg->vmstats->state更新数据



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



