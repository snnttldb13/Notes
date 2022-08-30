
# Standby Sunucularda Datafile Kurtarma

 Bazen standby sunucularda bir ya da daha fazla datafile dosyasında bozulmalar yaşanabilir. Disk hatası ya da kullanıcı hataları buna sebep olabilir. Böyle bir durumda
 bozulan datafile'ı kurtarmak için gerekli işlemler aşağıda anlatlıyor.
 
- Bozulan datafile'ın _primary_ sunucudan yedeği alınarak kurtarma çalışması yapılabilir. Bunun için ilk önce hangi datafile'ın bozuk olduğu tespit edilmelidir.
  Standby sunucu alert log'u incelenerek bozulan datafile tespit edilebilir.

```sh
tail -100 /u01/app/oracle/diag/rdbms/sby3/sby3/trace/alert_sby3.log
```
```
ORA-00600: internal error code, arguments: [3020], [81], [221146], [339959770], [], [], [], [], [], [], [], []
ORA-10567: Redo is inconsistent with data block (file# 81, block# 221146, file offset is 1811628032 bytes)
ORA-10564: tablespace UNDO2
ORA-01110: data file 81: '/u02/oradata/undotbs03.dbf'
ORA-10560: block type 'KTU UNDO BLOCK'
Tue Mar 23 09:27:02 2021
```


- Yapılan incelemeden anlaşıldığı üzere 81 numaralı datafile bozulmuş.
- Bozulan datafile'ı kurtarmak için _primary_ sunucudan RMAN aracı ile yedek alınır.

```sql
rman> backup datafile 81 format '/yedek/standbyicinyedek.bck';
```


- Alınan yedek dosyası standby sunucuya kopyalanır, bunun için _scp_ komutu kullanılabilir.

```sh
scp standbyicinyedek.bck sby3:/u03/yedek/
```


- Yedek dosyası standby sunucuda RMAN ile kataloglanır.

```sql
RMAN> catalog backuppiece '/u03/yedek/standbyicinyedek.bck';
```
```
cataloged backup piece
backup piece handle=/u03/yedek/standbyicinyedek.bck RECID=220 STAMP=1067939141
```


- Kataloglama işleminin ardından bozulan datafile RMAN aracı ile _restore_ edilir.

```sql
RMAN> restore datafile 81;
```
```
Starting restore at 23-MAR-21
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00081 to /u02/oradata/undotbs03.dbf
channel ORA_DISK_1: reading from backup piece /tmp/standbyicinyedek.bck
channel ORA_DISK_1: errors found reading piece handle=/tmp/standbyicinyedek.bck
channel ORA_DISK_1: failover to piece handle=/u03/yedek/standbyicinyedek.bck tag=TAG20210323T093813
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:35
Finished restore at 23-MAR-21
```


- Bu işlem ile birlikte kurtarma işlemleri tamamlanmış oluyor. Birden fazla datafile bozulmuş ise de bu yöntem kullanılabilir. 
- Replikasyon işlemlerinin devamı için MRP0 process'i başlatılır.

```sql
alter database recover managed standby database using current logfile disconnect;
```


- Herhangi bir hata olup olmadığın kontrolü için MRP0 processinin çalışmaya devam edip etmediği kontrol edilebilir.

```sql
select process,status from v$managed_standby;
```
```
PROCESS   STATUS
--------- ------------
ARCH      CLOSING
ARCH      CONNECTED
ARCH      CLOSING
ARCH      CLOSING
RFS       IDLE
MRP0      APPLYING_LOG
RFS       IDLE

7 rows selected.
```
