root@disk-test-0:/# fio --section=test6 jobs.fio
test6: (g=0): rw=write, bs=(R) 256KiB-256KiB, (W) 256KiB-256KiB, (T) 256KiB-256KiB, ioengine=libaio, iodepth=1
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [W(1)][100.0%][w=33.0MiB/s][w=132 IOPS][eta 00m:00s]
test6: (groupid=0, jobs=1): err= 0: pid=1353: Wed Feb 27 17:38:16 2019
  write: IOPS=131, BW=32.8MiB/s (34.4MB/s)(9832MiB/300004msec); 0 zone resets
    slat (usec): min=31, max=4069, avg=60.15, stdev=28.75
    clat (usec): min=6824, max=43226, avg=7561.45, stdev=920.92
     lat (usec): min=6868, max=43269, avg=7622.64, stdev=922.34
    clat percentiles (usec):
     |  1.00th=[ 6980],  5.00th=[ 7111], 10.00th=[ 7177], 20.00th=[ 7242],
     | 30.00th=[ 7308], 40.00th=[ 7373], 50.00th=[ 7373], 60.00th=[ 7439],
     | 70.00th=[ 7570], 80.00th=[ 7635], 90.00th=[ 7832], 95.00th=[ 8291],
     | 99.00th=[11338], 99.50th=[11863], 99.90th=[15270], 99.95th=[22938],
     | 99.99th=[38536]
   bw (  KiB/s): min=23040, max=35328, per=99.99%, avg=33554.28, stdev=1690.41, samples=600
   iops        : min=   90, max=  138, avg=131.04, stdev= 6.60, samples=600
  lat (msec)   : 10=97.74%, 20=2.19%, 50=0.06%
  cpu          : usr=0.40%, sys=0.85%, ctx=39353, majf=0, minf=1
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,39326,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=32.8MiB/s (34.4MB/s), 32.8MiB/s-32.8MiB/s (34.4MB/s-34.4MB/s), io=9832MiB (10.3GB), run=300004-300004msec

Disk stats (read/write):
  sdc: ios=0/39329, merge=0/9, ticks=0/295168, in_queue=4668, util=1.56%