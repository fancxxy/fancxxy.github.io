---
title: 计算Linux系统的CPU使用率
date: 2019-09-23 16:36:46
tags: Linux
---

Linux系统启动后会在内存中生成一个`/proc`虚拟文件系统，内部存储了内核运行状态，计算CPU使用率用到了其中的`/proc/stat`文件。
<!-- more -->

### /proc/stat
`/proc/stat`文件第一行是CPU的总数据，后续带`cpuN`前缀的行是单核的数据。
```sh
$ cat /proc/stat
cpu  162267 284 100504 121965754 7515 0 1107 0 0 0
cpu0 76115 139 51012 60979072 3208 0 688 0 0 0
cpu1 86152 144 49492 60986681 4306 0 419 0 0 0
intr 27522326 126 9 0 0 0 0 3 0 0 0 0 0 202 0 0 0 0 0 0 0 0 0 0 0 0 1415793 26 0 156 0 1613129 2 0 0 0 338829 0 72 0 25 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ctxt 46707394
btime 1594775312
processes 144328
procs_running 3
procs_blocked 0
softirq 25561160 0 9192916 694207 3030422 333841 0 554167 7982397 0 3773210
```

数据的单位是`jiffies`，表示的是系统自启动以来运行在不同状态时产生的中断次数，而每秒产生的中断次数是通过`USER_HZ`定义的，是一个固定值，Linux内核默认为1000，所以用`jiffies/USER_HZ`就能得到运行时间了。

列字段的含义可以通过`man proc`查看。
```
/proc/stat
              kernel/system statistics.  Varies with architecture.  Common entries include:

              cpu  3357 0 4313 1362393
                     The amount of time, measured in units of USER_HZ (1/100ths of  a  second  on
                     most  architectures,  use  sysconf(_SC_CLK_TCK)  to obtain the right value),
                     that the system spent in various states:

                     user   (1) Time spent in user mode.

                     nice   (2) Time spent in user mode with low priority (nice).

                     system (3) Time spent in system mode.

                     idle   (4) Time spent in the idle task.  This value should be USER_HZ  times
                            the second entry in the /proc/uptime pseudo-file.

                     iowait (since Linux 2.5.41)
                            (5) Time waiting for I/O to complete.

                     irq (since Linux 2.6.0-test4)
                            (6) Time servicing interrupts.

                     softirq (since Linux 2.6.0-test4)
                            (7) Time servicing softirqs.

                     steal (since Linux 2.6.11)
                            (8)  Stolen  time, which is the time spent in other operating systems
                            when running in a virtualized environment

                     guest (since Linux 2.6.24)
                            (9) Time spent running a virtual  CPU  for  guest  operating  systems
                            under the control of the Linux kernel.

                     guest_nice (since Linux 2.6.33)
                            (10)  Time spent running a niced guest (virtual CPU for guest operat‐
                            ing systems under the control of the Linux kernel).
```

### 使用率计算
计算CPU使用率，就是计算CPU除idle状态的其他时长之和占CPU总时长的比例，实现方法是在一个间隔时间内前后读两次`/proc/stat`文件，分别计算出两次的busy时间和total时间相除。
```
percent = 100 * (busy2 - busy1) / (total2 - total1)
```

定义`CPUTime`数据结构。
```go
// CPUTime cpu在各种状态下的运行时间累计值
type CPUTime struct {
	Name      string  `json:"name"`
	User      float64 `json:"user"`
	Nice      float64 `json:"nice"`
	System    float64 `json:"system"`
	Idle      float64 `json:"idle"`
	Iowait    float64 `json:"iowait"`
	Irq       float64 `json:"irq"`
	Softirq   float64 `json:"softrrq"`
	Steal     float64 `json:"steal"`
	Guest     float64 `json:"guest"`
	GuestNice float64 `json:"guestNice"`
}
```

`CPUTimes()`函数读取`/proc/stat`文件并解析以cpu开头的行数据，返回`[]*CPUTime`数组。
```go
// CPUTimes 计算CPUTime
func CPUTimes() ([]*CPUTime, error) {
	lines, err := utils.ReadLines("/proc/stat")
	if err != nil {
		return nil, err
	}

	var ret []*CPUTime
	for _, line := range lines {
		if !strings.HasPrefix(line, "cpu") {
			break
		}

		stat, err := parseStatLine(line)
		if err != nil {
			continue
		}
		ret = append(ret, stat)
	}

	return ret, nil
}

func parseStatLine(line string) (*CPUTime, error) {
	fields := strings.Fields(line)

	if len(fields) == 0 {
		return nil, errors.New("stat line contains no cpu data")
	}

	if strings.HasPrefix(fields[0], "cpu") == false {
		return nil, errors.New("stat line not start with \"cpu\"")
	}

	name := fields[0]

	user, err := strconv.ParseFloat(fields[1], 64)
	if err != nil {
		return nil, err
	}

	nice, err := strconv.ParseFloat(fields[2], 64)
	if err != nil {
		return nil, err
	}

	system, err := strconv.ParseFloat(fields[3], 64)
	if err != nil {
		return nil, err
	}

	idle, err := strconv.ParseFloat(fields[4], 64)
	if err != nil {
		return nil, err
	}

	iowait, err := strconv.ParseFloat(fields[5], 64)
	if err != nil {
		return nil, err
	}

	irq, err := strconv.ParseFloat(fields[6], 64)
	if err != nil {
		return nil, err
	}

	softirq, err := strconv.ParseFloat(fields[7], 64)
	if err != nil {
		return nil, err
	}

	steal, err := strconv.ParseFloat(fields[8], 64)
	if err != nil {
		return nil, err
	}

	guest, err := strconv.ParseFloat(fields[9], 64)
	if err != nil {
		return nil, err
	}

	// since Linux 2.6.33
	guestNice, err := strconv.ParseFloat(fields[10], 64)
	if err != nil {
		return nil, err
	}

	return &CPUTime{
		Name:      name,
		User:      user / hertZ,
		Nice:      nice / hertZ,
		System:    system / hertZ,
		Idle:      idle / hertZ,
		Iowait:    iowait / hertZ,
		Irq:       irq / hertZ,
		Softirq:   softirq / hertZ,
		Steal:     steal / hertZ,
		Guest:     guest / hertZ,
		GuestNice: guestNice / hertZ,
	}, nil
}
```

`Percent()`函数接收一个时间段作为采样间隔，使用上面提到的公式计算CPU使用率。
```go
// Percent 获取所有核的cpu使用率
func Percent(interval time.Duration) ([]float64, error) {
	t1, err := CPUTimes()
	if err != nil {
		return nil, err
	}

	time.Sleep(interval)

	t2, err := CPUTimes()
	if err != nil {
		return nil, err
	}

	return calCPUPercent(t1, t2)
}

func calCPUPercent(time1, time2 []*CPUTime) ([]float64, error) {
	if len(time1) != len(time2) {
		return nil, errors.New("structure of both cpu time are not equal")
	}

	ret := make([]float64, 0, len(time1))

	for i := range time2 {
		total1 := time1[i].Total()
		busy1 := total1 - time1[i].Idle

		total2 := time2[i].Total()
		busy2 := total2 - time2[i].Idle

		ret = append(ret, math.Min(100, math.Max(0, (busy2-busy1)/(total2-total1)*100)))
	}
	return ret, nil
}
```

设置采样周期为10秒。
```go
func main() {
    percent, _ := cpu.Percent(10 * time.Second)
    fmt.Printf("percent: %f%%\n", percent[0])   
}
```

查看计算结果
```
percent: 0.050075%
```

为了支持连续固定的采集输出计算结果，可以存储上一次的CPU数据，调用`Percent()`函数时如果传递的参数小于等于0直接用当前采集的结果和存储的历史数据直接计算，最后更新历史数据。
```go
type safeCPUTimes struct {
	sync.Mutex
	cpuTimes []*CPUTime
}

var lastCPUTimes safeCPUTimes

func init() {
	lastCPUTimes.Lock()
	defer lastCPUTimes.Unlock()
	lastCPUTimes.cpuTimes, _ = CPUTimes()
}

// Percent 获取所有核的cpu使用率
func Percent(interval time.Duration) ([]float64, error) {
	var t1, t2 []*CPUTime

	cpuTimes, err := CPUTimes()
	if err != nil {
		return nil, err
	}

	if interval <= 0 {
		lastCPUTimes.Lock()
		t1 = lastCPUTimes.cpuTimes
		t2 = cpuTimes
		lastCPUTimes.cpuTimes = cpuTimes
		lastCPUTimes.Unlock()
	} else {
		t1 = cpuTimes
		time.Sleep(interval)
		t2, err = CPUTimes()
		if err != nil {
			return nil, err
		}
	}

	return calCPUPercent(t1, t2)
}
```

设置采样周期为10秒并查看计算结果。
```go
func main() {
	ticker := time.NewTicker(time.Second * 10)
	for range ticker.C {
		percent, _ := cpu.Percent(0)
		fmt.Printf("percentage: %f%%\n", percent[0])
	}
}
```

```
percentage: 0.100100%
percentage: 0.100050%
percentage: 0.050050%
percentage: 0.100100%
percentage: 0.050050%
percentage: 0.100100%
percentage: 0.100050%
percentage: 0.050075%
```

完整代码可以在[GitHub](https://github.com/fancxxy/metrics)查看。
