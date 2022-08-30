
 # ISCSI Disk Bağlama
 
  Çoğu zaman yedekleme işlemleri için kullanmak üzere ISCSI LUN'lar kullanılabiliyor. Aşağıda bir ISCSI diski bir linux sunucusuna bağlamak için gerekli işlem adımları
  tanımlanıyor.
  
  

- Sunucuda _iscsiadm_ paketi yüklenmemiş ise ilk olarak bu paket kurulur.

```sh
 yum install iscsi-initiator-utils -y 
```


- ISCSI disk bir kullanıcı adı parola kullanılarak bağlanılıcak ise öncelikle ilgili ayar dosyasında düzenlemeler yapılmalıdır. _iscsid.conf_ dosyasında aşağıdaki 
 değişiklikler yapılır.
 
```sh
nano /etc/iscsi/iscsid.conf
```
```
 Aşağıdaki satırlar yorum satırı olmaktan çıkarılır.
 
 # To enable CHAP authentication set node.session.auth.authmethod
 # to CHAP. The default is None.
 node.session.auth.authmethod = CHAP
```
```
Verilen kullanıcı adı parola bilgisi girilerek dosya kaydedilir.

# To set a CHAP username and password for initiator
# authentication by the target(s), uncomment the following lines:
node.session.auth.username = hbys
node.session.auth.password = 1q2w3e4r
```


- IP üzerinden LUN taraması yapılır.

```sh
 iscsiadm -m discovery -t sendtargets -p 10.1.1.2
```


- Çıkan sonuca göre iqn adresi ile disk bağlanır. 

```sh
 iscsiadm -m node -T iqn.2000-01.com.synology:Synology.Target-1.0f6b76b028 \ -p 10.1.1.2:3260 -l
```


- Aşağıdaki komut ile herhangi bir ISCSI diski bağlı mı kontrol edilebilir.

```sh
 iscsiadm -m session
```


- Mevcut ISCSI oturumlarını sonlandırmak için aşağıdaki komut kullanılır.

```sh
 iscsiadm -m node -u
```	
	
