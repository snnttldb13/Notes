
 # HugePages Ayarları
 
 Linux sunucularda RAM miktarı 64Gb ve üzeri ise HugePages'in aktif edilerek kullanılması sunucu performansını ciddi derecede artırmaktadır. Aşağıda
 hugepage ayarlarının yapılması ve durum kontrolü için gerekli yönergeler mevcut.
 
 
   ### Kernel versiyon 4.14 ve 4.14'ten küçük ise

- Mevcut durumda hugepage ayarlarını kontrol etmek için aşağıdaki komut çalıştırılır. _HugePages_Total_  değeri 0 olarak görünüyor, hugepages açılmamış.

```sh
 grep Huge /proc/meminfo
```
```
AnonHugePages:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```


- Gereken hugepage değerini hesaplamak için _/home/oracle/_ dizinine  _hugepages_setting.sh_ isminde bir dosya oluşturulur ve içine aşağıdakiler girilir.
   
```sh
 nano /home/oracle/hugepages_setting.sh
```
```
#!/bin/bash
#
# hugepages_settings.sh
#
# Linux bash script to compute values for the
# recommended HugePages/HugeTLB configuration
#
# Note: This script does calculation for all shared memory
# segments available when the script is run, no matter it
# is an Oracle RDBMS shared memory segment or not.
# Check for the kernel version
KERN=`uname -r | awk -F. '{ printf("%d.%d\n",$1,$2); }'`
# Find out the HugePage size
HPG_SZ=`grep Hugepagesize /proc/meminfo | awk {'print $2'}`
# Start from 1 pages to be on the safe side and guarantee 1 free HugePage
NUM_PG=1
# Cumulative number of pages required to handle the running shared memory segments
for SEG_BYTES in `ipcs -m | awk {'print $5'} | grep "[0-9][0-9]*"`
do
   MIN_PG=`echo "$SEG_BYTES/($HPG_SZ*1024)" | bc -q`
   if [ $MIN_PG -gt 0 ]; then
      NUM_PG=`echo "$NUM_PG+$MIN_PG+1" | bc -q`
   fi
done
# Finish with results
case $KERN in
   '2.4') HUGETLB_POOL=`echo "$NUM_PG*$HPG_SZ/1024" | bc -q`;
          echo "Recommended setting: vm.hugetlb_pool = $HUGETLB_POOL" ;;
   '2.6' | '3.8' | '3.10' | '4.1' | '4.14' ) echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    *) echo "Unrecognized kernel version $KERN. Exiting." ;;
esac
# End
```


- Ardından yeni oluşturulan dosyaya çalıştırma izni verilir ve çalıştırılır. Optimum değerin hesaplanabilmesi için sunucu yük altında iken çalıştırılmalıdır.

```sh
chmod u+x hugepages_setting.sh
```
```sh
./hugepages_setting.sh
```
```
Recommended setting: vm.nr_hugepages = 305 
```


- Bir önceki adımda bulunan değer 1 (bir) artırılarak aşağıdaki dosyanın sonuna eklenir.

```sh
nano /etc/sysctl.conf
```


- Ayarların devreye alınabilmesi için sunucu _reboot_ edilmelidir. Sunucu yeniden başladıktan sonra hugepage ayarları kontrol edilebilir.

```sh
grep Huge /proc/meminfo
```
```
AnonHugePages:         0 kB
HugePages_Total:     306
HugePages_Free:      306
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```


   ### Kernel versiyon 4.14'ten büyük ise


- Mevcut durumda hugepage ayarlarını kontrol etmek için aşağıdaki komut çalıştırılır. _HugePages_Total_  değeri 0 olarak görünüyor, hugepages açılmamış.

```sh
 grep Huge /proc/meminfo
```
```
AnonHugePages:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```


- Gereken hugepage değerini hesaplamak için _/home/oracle/_ dizinine  _hugepages_setting.sh_ isminde bir dosya oluşturulur ve içine aşağıdakiler girilir.
   
```sh
 nano /home/oracle/hugepages_setting.sh
```
```
#!/bin/bash
#
# hugepages_settings.sh
#
# Linux bash script to compute values for the
# recommended HugePages/HugeTLB configuration
# on Oracle Linux
#
# Note: This script does calculation for all shared memory
# segments available when the script is run, no matter it
# is an Oracle RDBMS shared memory segment or not.
#
# This script is provided by Doc ID 401749.1 from My Oracle Support
# http://support.oracle.com

# Welcome text
echo "
This script is provided by Doc ID 401749.1 from My Oracle Support
(http://support.oracle.com) where it is intended to compute values for
the recommended HugePages/HugeTLB configuration for the current shared
memory segments on Oracle Linux. Before proceeding with the execution please note following:
 * For ASM instance, it needs to configure ASMM instead of AMM.
 * The 'pga_aggregate_target' is outside the SGA and
   you should accommodate this while calculating the overall size.
 * In case you changes the DB SGA size,
   as the new SGA will not fit in the previous HugePages configuration,
   it had better disable the whole HugePages,
   start the DB with new SGA size and run the script again.
And make sure that:
 * Oracle Database instance(s) are up and running
 * Oracle Database 11g Automatic Memory Management (AMM) is not setup
   (See Doc ID 749851.1)
 * The shared memory segments can be listed by command:
     # ipcs -m


Press Enter to proceed..."

read

# Check for the kernel version
KERN=`uname -r | awk -F. '{ printf("%d.%d\n",$1,$2); }'`

# Find out the HugePage size
HPG_SZ=`grep Hugepagesize /proc/meminfo | awk '{print $2}'`
if [ -z "$HPG_SZ" ];then
    echo "The hugepages may not be supported in the system where the script is being executed."
    exit 1
fi

# Initialize the counter
NUM_PG=0

# Cumulative number of pages required to handle the running shared memory segments
for SEG_BYTES in `ipcs -m | cut -c44-300 | awk '{print $1}' | grep "[0-9][0-9]*"`
do
    MIN_PG=`echo "$SEG_BYTES/($HPG_SZ*1024)" | bc -q`
    if [ $MIN_PG -gt 0 ]; then
        NUM_PG=`echo "$NUM_PG+$MIN_PG+1" | bc -q`
    fi
done

RES_BYTES=`echo "$NUM_PG * $HPG_SZ * 1024" | bc -q`

# An SGA less than 100MB does not make sense
# Bail out if that is the case
if [ $RES_BYTES -lt 100000000 ]; then
    echo "***********"
    echo "** ERROR **"
    echo "***********"
    echo "Sorry! There are not enough total of shared memory segments allocated for
HugePages configuration. HugePages can only be used for shared memory segments
that you can list by command:

    # ipcs -m

of a size that can match an Oracle Database SGA. Please make sure that:
 * Oracle Database instance is up and running
 * Oracle Database 11g Automatic Memory Management (AMM) is not configured"
    exit 1
fi

# Finish with results
case $KERN in
    '2.4') HUGETLB_POOL=`echo "$NUM_PG*$HPG_SZ/1024" | bc -q`;
           echo "Recommended setting: vm.hugetlb_pool = $HUGETLB_POOL" ;;
    '2.6') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    '3.8') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    '3.10') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    '4.1') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    '4.14') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    '4.18') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    '5.4') echo "Recommended setting: vm.nr_hugepages = $NUM_PG" ;;
    *) echo "Kernel version $KERN is not supported by this script (yet). Exiting." ;;
esac

# End
```


- Ardından yeni oluşturulan dosyaya çalıştırma izni verilir ve çalıştırılır. Optimum değerin hesaplanabilmesi için sunucu yük altında iken çalıştırılmalıdır.

```sh
chmod u+x hugepages_setting.sh
```
```sh
./hugepages_setting.sh
```
```
Recommended setting: vm.nr_hugepages = 305 
```


- Bir önceki adımda bulunan değer 1 (bir) artırılarak aşağıdaki dosyanın sonuna eklenir.

```sh
nano /etc/sysctl.conf
```


- Ayarların devreye alınabilmesi için sunucu _reboot_ edilmelidir. Sunucu yeniden başladıktan sonra hugepage ayarları kontrol edilebilir.

```sh
grep Huge /proc/meminfo
```
```
AnonHugePages:         0 kB
HugePages_Total:     306
HugePages_Free:      306
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```   
   
   
