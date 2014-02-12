---
layout: post
title:  "Создание виртуальной машины для локальной работы с нескольких компьютеров"
date:   2014-02-11 22:51:21
categories: ru programming
tags: virtualbox ubuntu php
---
У меня есть два компьютера (десктоп W7 и лаптоп Mac), и я часто переключаюсь между ними - будь то в поездке или на диване перед телевизором. Для этого мне потребовалось настроить рабочее окружение так, чтобы переключение между машинами было как можно более легким.


Вот несколько задач, которые требовалось решить:

* Сохранение текущего состояния всех проектов - базы данных, конфигурационных файлов, бутстрапов, установленных расширений и т.д.
* Синхронизация кода всех проектов
* Возможность пользоваться всем этим в поездке

##Сохранение текущего состояния

Сложно переоценить все преимущества, которые дает вынесение всей серверной части (apache, nginx, php, nodejs, etc) на линуксовую машину. А виртуальная линукс машина дает еще несколько преимуществ - в том числе решение описываемых здесь задач.

Добиться того, что мы увидим ниже, можно частично с [Vargant](http://docs.vagrantup.com/v2/why-vagrant/index.html). А можно и объединить подходы - они друг другу не противоречат.

Здесь и в дальнейшем я подразумеваю, что вы знакомы с работой в командной строке, и что в случае необходимости вы сами разберетесь с непонятными вам вещами, и что всю основную настройку необходимых сервисов вы либо можете проделать сами, либо уже проделали. Если вы не знакомы с линуксом, рекомендую начать [здесь](http://www.ubuntu.com/server). Для своих целей я использую [VirtualBox](https://www.virtualbox.org)/[Ubuntu](http://www.ubuntu.com/server), но все то же самое наверняка можно проделать в любой другой сборке.

Итак. Первое, что мы настроим - это сеть.
Виртуальная машина должна иметь доступ в интернет, в домашней обстановке должна быть доступна для обоих компьютеров, а в поездке только для ноутбука.
Редактируем `/etc/network/interfaces`:

{% highlight bash %}
# NAT adapter. Предоставляет доступ к интернет
auto eth0
iface eth0 inet dhcp

# Host-only adapter; приватная сеть между гостевой и родительской машинами
auto eth1
iface eth1 inet static
	address 192.168.56.XXX
	netmask 255.255.255.0
	gateway 192.168.56.100

# Bridged adapter; локальная сеть
auto eth2
iface eth2 inet static
	address 192.168.1.XXX
	netmask 255.255.255.0
	gateway 192.168.1.1
{% endhighlight %}

XXX и gateway выбирайте сами, но учитывайте настройки VirtualBox: `Preferences > Network > Host-only networks -> edit -> DHCP server`. На десктопе в настройках виртуальной машины создаем все три адаптера.

Теперь мы можем выключить виртуальную машину и скопировать жесткий диск (vdi файл) на флешку. В принципе, его можно всегда хранить на флешке.

На лаптопе создаем новую виртуальную машину (старайтесь сохранять похожие настройки) и выбираем существующий жесткий диск. В этой виртуальной машине нам нужно создать только два сетевых адаптера - NAT и Host-only. (В принципе, можно на обоих машинах обойтись всего одним - но это рискованно).

Запускаем машину на лаптопе. Если она очень долго запускается и пишет сообщения вроде "waiting 60 seconds for network configuration", то редактируем файл `/etc/init/failsafe.conf`. В нем можно уменьшить таймаут - например, до одной секунды.

Теперь виртуальная машина может работать на десктопе (192.168.1.XXX - доступна обоим компьютерам) или на лаптопе (192.169.56.ХХХ - доступна только лаптопу).

##Доступ к коду

Как и в Vargant, сам код проектов находится на host машине. Чтобы виртуальная машина имела к нему доступ, нужно монтировать папки из хоста в гостевую машину. (В VirtualBox есть механизм создания общей папки - наверное можно воспользоваться им, не проверял. Vargant использует именно ее).

{% highlight bash %}
$ sudo apt-get install cifs.utils
$ sudo apt-get install sshfs
{% endhighlight %}

С такого рода папками есть несколько проблем. При изменении файла на виндос хост-машине в линуксе не срабатывают inotify события, из-за чего страдают сервисы, следящие за изменениями в папках (например, iwatch). Частично эту проблему решает [watchntouch](https://github.com/rubyruy/watchntouch).
Еще одна проблема в том, что apache отказывается нормально работать, когда document root  находится внутри смонтированной папки. Поэтому мы создаем document root на гостевой машине, а внутри нее симлинки на все нужные папки внутри проекта.

Для этого я использую php скрипт (такой же можно написать на любом языке), который помимо этого переключает виртуальную машину с десктопа на лаптоп и обратно.

{% highlight php %}
<?php
$profile	= $argv[1];
$ip			= isset($argv[2]) ? $argv[2] : null;

if (empty($profile) || empty($ip)) {
	die("Please specify profile and ip");
}

$profiles	= array(
	'w7'	=> array(
		'mountpoint'	=> '/media/w7',
		'mount'			=> 'mount.cifs //'.$ip.'/projects {mountpoint} -o username=user,password=password,uid=user,gid=group,file_mode=0777,dir_mode=0777,iocharset=utf8,nodfs',
		'unmount'		=> 'umount {mountpoint}'
	),
	'osx'	=> array(
		'mountpoint'	=> '/media/osx',
		'mount'			=> 'sshfs kuindji@'.$ip.':/Users/kuindji/Documents/projects {mountpoint} -o allow_other',
		'unmount'		=> 'fusermount -u {mountpoint}'
	)
);
$links  	=  array(
	'/path/to/docroot/folder1' 	=> '{mountpoint}/project/folder1',
	'/path/to/docroot/folder2' 	=> '{mountpoint}/project/folder2'
	// etc
);
$services 	= array(
	"php-fastcgi",
	"nginx"
);

if (!isset($profiles[$profile])) {
	die("Profile not found");
}

$mountpoint	= $profiles[$profile]['mountpoint'];

echo "-- Stopping services...\n";
foreach ($services as $service) {
	passthru("service $service stop");
}

echo "-- Unlinking previous\n";
foreach ($link as $local => $remote) {
	passthru("unlink ".$local);
}

echo "-- Unmounting all previous mounts\n";
foreach ($profiles as $name => $params) {
	passthru(str_replace(
		'{mountpoint}',
		$params['mountpoint'],
		$params['unmount']
	));
}

$mp = $profiles[$profile]['mountpoint'];

echo "-- Mounting profile\n";
passthru(str_replace(
	'{mountpoint}',
	$mp,
	$profiles[$profile]['mount']
));

echo "-- Creating links\n";
foreach ($links as $local => $remote) {
	passthru("ln -s ".str_replace('{mountpoint}', $mp, $remote)." $local");
}

echo "-- Starting services...\n";
$services = array_reverse($services);
foreach ($services as $service) {
	passthru("service $service start");
}
?>
{% endhighlight %}

Можно вызывать этот скрипт вручную после старта машины, а можно запускать его автоматически при загрузке.

{% highlight bash %}
$ sudo php profile.php w7 192.168.1.2
{% endhighlight %}

Таким образом мы можем переключать виртуальную машину между десктопом и лаптопом. Осталось синхронизировать файлы всех проектов.

##Синхронизация проектов

{% highlight php %}
<?php
$opt 	= getopt("", array(
	"from:", "to:", "osx:", "w7:",
	"dry::", "out::", "path::",
	"help::", "initial::"
));
extract($opt);

if (!empty($help)) {
	echo "Usage:\n";
	echo "--from	profile to sync from\n";
	echo "--to 		profile to sync to\n";
	echo "--w7		w7 ip if needed\n";
	echo "--osx		osx ip\n";
	echo "--dry		do not apply changes\n";
	echo "--out		path to store rsync output in\n";
	echo "--path	path to sync; relative to 'projects'\n";
	echo "--initial	initial upload\n";
	exit;
}

$profiles	= array(
	"w7"	=> array(
		'mountpoint'	=> '/media/w7',
		'mount'			=> "mount.cifs //{$w7}/projects {location} ".
						"-o username=ubuntu,password=Gfhjkm,uid=kuindji,gid=kuindji,".
						"file_mode=0777,dir_mode=0777,iocharset=utf8,nodfs",
		'unmount'		=> 'umount {location}'
	),
	"osx"	=> array(
		'mountpoint'	=> '/media/osx',
		'mount'			=> "sshfs kuindji@{$osx}:/Users/kuindji/Documents/projects {location} -o allow_other",
		'unmount'		=> 'fusermount -u {location}'
	)
);

$tmpFrom		= substr(md5(microtime(true)), 0, 10);
$tmpTo 			= substr(md5(microtime(true)), 10, 10);
$fromLocation	= "/tmp/$tmpFrom";
$toLocation		= "/tmp/$tmpTo";

if (file_exists($fromLocation)) {
	die("$fromLocation exists");
}
if (file_exists($toLocation)) {
	die("$toLocation exists");
}

echo "-- creating directories\n";
mkdir($fromLocation);
chmod($fromLocation, 0777);
mkdir($toLocation);
chmod($toLocation, 0777);

function mount($profile, $location) {
	global $profiles;

	$cmd	= $profiles[$profile]['mount'];
	$cmd 	= str_replace('{location}', $location, $cmd);
	echo "-- $cmd\n";
	passthru($cmd);
}

function unmount($profile, $location) {
	global $profiles;

	$cmd	= $profiles[$profile]['unmount'];
	$cmd 	= str_replace('{location}', $location, $cmd);
	echo "-- $cmd\n";
	passthru($cmd);
}

echo "-- mounting locations\n";
mount($from, $fromLocation);
mount($to, $toLocation);


$rsyncFlags     = "-rLhtz --delete --force --modify-window=5 ";
$rsyncFlags		.= "--exclude='.idea/*' ";
$rsyncFlags 	.= "--itemize-changes --out-format='%i %f' ";
$rsyncFlags 	.= "--filter='-p .DS_Store' ";

if (!empty($dry)) {
	$rsyncFlags	.= " --dry-run";
}

if (empty($path)) {
	$cmd 		= "rsync $rsyncFlags $fromLocation/ $toLocation/";
}
else {
	$cmd 		= "rsync $rsyncFlags $fromLocation/$path $toLocation/$path";
}

if (!empty($out)) {
	$cmd 		.= " > $out";
}

echo "-- syncing: $cmd\n";
passthru($cmd);

echo "-- unmounting locations\n";
unmount($from, $fromLocation);
unmount($to, $toLocation);

echo "-- removing directories\n";
passthru("rm -r $fromLocation");
passthru("rm -r $toLocation");
?>
{% endhighlight %}

Этот скрипт выполняется на виртуальной машине. Он монтирует обе папки проектов к себе и синхронизирует их с помощью rsync. Таким образом переключение между компьютерами выполняется двумя командами:

{% highlight bash %}
$ sudo project-sync --from=w7 --to=osx --w7=w7IP --osx=osxIP
$ sudo profile osx osxIP
{% endhighlight %}
