
# UNDO Tablespace'ini Değiştirme

 Bazı durumlarda UNDO tablespace'ini değiştirmek gerekebilir. Örneğin veritabanı kendiliğinden kapanıyorsa undo tablespace'i değiştirilebilir.
 Bunun için gerekli işlemler aşağıda anlatılmıştır.
 

- Veritabanı kapatılır, nomount modda geri açılır

```sql
 shu immediate
 startup nomount
```


- UNDO tablespace'i üzerinde işlem yapabilmek için _undo_management_ parametresi  _manuel_  olarak ayarlanır.

```sql
 alter system set undo_management = manual scope=spfile;
```


- Veritabanı kapatılır ve normal olarak açılır.

```sql
 shu immediate
 startup
```


- Yeni bir UNDO tablespace'i oluşturulur. Bunun için _sqlplus_ aracı kullanılabilir.

```sql
CREATE UNDO TABLESPACE UNDOTBS02 DATAFILE '/c_4/oradata/orcl/undotbs_02.dbf' SIZE 300M AUTOEXTEND ON;
```


- Ardından default undo tablespace'i olarak yeni oluşturduğumuz tablespace seçilir. Bunun için _sqlplus_ aracı kullanılabilir. Değişiklik olmazsa eğer _scope_ verilir.

```sql
 ALTER SYSTEM SET UNDO_TABLESPACE = UNDOTBS02;  
```


- Ardından _undo_management_ parametresi tekrar _auto_ olarak ayarlanır.

```sql
 alter system set undo_management = auto scope=spfile;
```


- Veritabanı kapatılır ve tekrar startup verilir

```sql
 shu immediate
 startup
```

- Ardından, istenirse eğer  eski undo tablespace'i drop edilebilir.

