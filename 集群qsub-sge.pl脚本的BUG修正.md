# 集群qsub-sge.pl脚本的BUG修正

## 一、BUG说明

### 范围：

	1. 涉及qsub-sge.pl投递任务的所有现有流程与任务。
	2. 部分“姓名全拼过长”的员工。

### 现象：

	qsub-sge.pl投递任务后，任务尚未完成，脚本“不正常”的判定该任务已结束，若设定了 `-reqsub` 参数，则会导致任务无限重投。

## 二、原因

### 定位：

	部分员工“姓名全拼过长”，`qstat`中user信息列的姓名显示不全，命令 `whoami` 和 `qstat` 获得的uesr ID不匹配，导致脚本判定出错。
### 详细描述：

qsub-sge.pl脚本中控制检测运行任务数的子函数`run_count`如下：


```
sub run_count {
	my $all_p = shift;
	my $run_p = shift;
	my $run_num = 0;
	
	%$run_p = ();
	my $user = `whoami`; chomp $user;
	my $qstat_result = `qstat -u $user`;
	if ($qstat_result =~ /failed receiving gdi request/) {
		$run_num = -1;
		return $run_num; 
	}
	my @jobs = split /\n/,$qstat_result; 
	foreach my $job_line (@jobs) {
		$job_line =~s/^\s+//;
		my @job_field = split /\s+/,$job_line;
		next if($job_field[3] ne $user);  ###BUG所在位置，由于qstat姓名显示不全，导致$job_field[3]和$user不匹配，所有仍在运行的任务都跳过了后续的判定，导致$run_num判定为0，即产生“所有任务已结束”的假象。###
		if (exists $all_p->{$job_field[0]}){
			
			my %died;
			died_nodes(\%died); 
			my $node_name = $1 if($job_field[7] =~ /(compute-\d+-\d+)/);
			if ( !exists $died{$node_name} && ($job_field[4] eq "qw" || $job_field[4] eq "r" || $job_field[4] eq "t") ) {  
				$run_p->{$job_field[0]} = $job_field[2]; ##job id => shell file name
				$run_num++;
			}else{
				`qdel $job_field[0]`;
			}
		}
	}
	
	return $run_num;
}
```


​	
​		

## 三、修复

### 修复办法:

命令 `qstat -j jobID |grep owner` 获得完整user ID，避免“姓名过长导致显示不全”的问题。

### 修复后:

```
sub run_count {
	my $all_p = shift;
	my $run_p = shift;
	my $run_num = 0;

	%$run_p = ();
	my $user = `whoami`; chomp $user;
	my $qstat_result = `qstat -u $user`;
	if ($qstat_result =~ /failed receiving gdi request/) {
		$run_num = -1;
		return $run_num; ##系统无反应
	}
	my @jobs = split /\n/,$qstat_result; 
	foreach my $job_line (@jobs) {
		$job_line =~s/^\s+//;
		my @job_field = split /\s+/,$job_line;
		my $owner=`qstat -j $job_field[0]|grep owner`;##add by ykc,20161031;
		$owner=~/owner:\s+(\w+)/;##add by ykc,20161031;
		my $user_qstat=$1;##add by ykc,20161031;
		#next if($job_field[3] ne $user);
		next if($user_qstat ne $user);##add by ykc,20161031;
		if (exists $all_p->{$job_field[0]}){
			
			my %died;
			died_nodes(\%died); ##the compute node is down, 有的时候节点已死，但仍然是正常状态
			my $node_name = $1 if($job_field[7] =~ /(compute-\d+-\d+)/);
			if ( !exists $died{$node_name} && ($job_field[4] eq "qw" || $job_field[4] eq "r" || $job_field[4] eq "t") ) {  
				$run_p->{$job_field[0]} = $job_field[2]; ##job id => shell file name
				$run_num++;
			}else{
				`qdel $job_field[0]`;
			}
		}
	}

	return $run_num; ##qstat结果中的处于正常运行状态的任务，不包含那些在已死掉节点上的僵尸任务
}
```

======

*2016.11.01 yukaicheng*