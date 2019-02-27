root@disk-test-0:/# fio --section=test10 jobs.fio
test10: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=16
fio-3.12
Starting 1 thread
Jobs: 1 (f=1): [w(1)][100.0%][w=69.8MiB/s][w=17.9k IOPS][eta 00m:00s]
test10: (groupid=0, jobs=1): err= 0: pid=1447: Wed Feb 27 18:14:39 2019
  write: IOPS=8291, BW=32.4MiB/s (33.0MB/s)(9717MiB/300007msec); 0 zone resets
    slat (usec): min=2, max=1509.3k, avg=117.20, stdev=2301.48
    clat (usec): min=6, max=1524.0k, avg=1802.27, stdev=8850.61
     lat (usec): min=74, max=1524.0k, avg=1919.93, stdev=9130.27
    clat percentiles (usec):
     |  1.00th=[   80],  5.00th=[   82], 10.00th=[   85], 20.00th=[   88],
     | 30.00th=[   91], 40.00th=[   94], 50.00th=[   98], 60.00th=[  103],
     | 70.00th=[  113], 80.00th=[  129], 90.00th=[11338], 95.00th=[15139],
     | 99.00th=[23200], 99.50th=[27919], 99.90th=[40109], 99.95th=[40109],
     | 99.99th=[43779]
   bw (  KiB/s): min= 2056, max=646376, per=100.00%, avg=33669.10, stdev=89092.42, samples=591
   iops        : min=  514, max=161594, avg=8417.26, stdev=22273.11, samples=591
  lat (usec)   : 10=0.01%, 100=55.09%, 250=33.70%, 500=0.49%, 750=0.05%
  lat (usec)   : 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 10=0.14%, 20=9.09%, 50=1.42%
  lat (msec)   : 500=0.01%, 750=0.01%, 1000=0.01%
  cpu          : usr=1.33%, sys=5.21%, ctx=17840, majf=0, minf=1
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=100.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.1%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2487477,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=16

Run status group 0 (all jobs):
  WRITE: bw=32.4MiB/s (33.0MB/s), 32.4MiB/s-32.4MiB/s (33.0MB/s-33.0MB/s), io=9717MiB (10.2GB), run=300007-300007msec

Disk stats (read/write):
  sdc: ios=0/328830, merge=0/350, ticks=0/309574384, in_queue=312233940, util=98.54%