
# Block Corruption

Oracle da block bozulmasından şüpheleniyorsak aşağıdaki komut ile herhangi bir block bozulması olup olmadığını kontrol edebiliriz.

```
SQL> select * from V$DATABASE_BLOCK_CORRUPTION;

``` 

Eğer sorgu sonucunda hernagi bir bozulmuş block bilgisi dönüyor ise aşağıdaki sorgu ile bozulan blockta ne tür bir datanın olduğunu kontrol edebiliriz.


```
SQL> select 
      relative_fno, 
      owner, 
      segment_name, 
      segment_type
    from 
      dba_extents
    where 
      file_id = 6
    and 
      86309 between block_id and block_id + blocks - 1;
      
``` 

Bozulan blockta index varsa eğer ilgili indeks silinip yeniden oluşturulabilir. Tablo var ise eğer ve öncesinden alınmış RMAN yedeği varsa block tamiri denenebilir.
