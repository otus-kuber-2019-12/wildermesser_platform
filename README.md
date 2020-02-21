# wildermesser_platform
wildermesser Platform repository

## Поды в kubernetes
В директории `kubernetes-intro` находятся манифесты для:
 - создания пода с примитивным http сервером. Содержимое генерируется в `init` контейнере
 - создания пода с frontend частью проекта Hipster-Shop

## Конроллеры в kubernetes
В директории `kubernetes-controllers` находятся манифесты для:
 - создания ReplicaSet для `frontend` и `paymentservice`
 - создания Deployment для `paymentservice` с различными стратегиями деплоя
 - создания Daemonset для `node-exporter` запускающего поды в том числе на `control-plane` нодах
## RBAC
В директории `kubernetes-security` находятся манифесты для различных политик управления доступом к k8s api
## Networks
В директории `kubernetes-networks` находятся манифесты для различных типов сервисов:
 - ClusterIP (обычный и headless)
 - LoadBalancer
Также манифест для Ingress.
В директории `kubernetes-networks/coredns` манифесты серисов типа LoadBalancer для публикации CoreDNS на одном и том же
адресе балансировщика Metallb
В директории `kubernetes-networks/dashboard` манифесты для публикации kubernetes-dashboard через Ingress
В директории `kubernetes-networks/canary` манифесты для Canary deployment с помощью Ingress
## Volumes
В директории `kubernetes-volumes` находятся манифесты для деплоя `minio` как `statefulset`, причём данные для авторизации
сохранены в ресурсе `secret` 
## Templating
В директории `kubernetes-templates` находятся файлы для шаблонизации манифестов различными способами:
 - helm
 - jsonnet
 - kustomize
## Operators
В директории `kubernetes-operators` находится реализация простого MySQL оператора, который обрабатывае события создания
и удаления CRD mysqls.otus.homework.
При создании ресурсе проиходит попытка восстановления из бэкапа. При удалении - создание бэкапа/
Пример:
```
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           1s         16m
restore-mysql-instance-job   1/1           44s        72s
```
После создания CR с теми же параметрами проиходит восстановление из бэкапа и в таблице есть данные:
```
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```
В случае успешного создания, в статусе CR появляется запись вида:
```
Status:                                                                                                                                                    │
  Kopf:                                                                                                                                                    │
    Dummy:  2020-02-10T07:55:01.509237                                                                                                                     │
  mysql_on_create:                                                                                                                                         │
    Message:   mysql-instance created without restore-job    
```
Это реализуется возвращением словаря в функции-обработчике mysql_on_update

В случае, если меняется пароль к базе данных, то:
1. С помощью `list_namespaced_pod('default',label_selector='app=%s' % name)` находится нужный под с mysql
2. С помощью `kubernetes.stream` в этоп поде запускается команда для смены пароля `mysqladmin -u root -p%s password %s' % (old_password, password)`
3. Обновляется деплоймент для синхронизации переменных окружения (в данном случае MYSQL_ROOT_PASSWORD применятеся только при первом старте, но в общем случае логично обновлять весь деплоймент)
4. С помощью `kubernetes.client.V1Scale` осуществляется скалирование пода в 0, а потом в 1 для применения новых значений переменных окружения
5. В `describe mysql` добавляется статус:
```
  mysql_on_update:                                                                                                                                         │
    Message:  password updated
```
## Vault
После установки Vault из helm чарта:
```
NAME: vault
LAST DEPLOYED: Thu Feb 20 11:28:05 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:
```
После инициализации:
```
/ $ vault operator init -key-shares=1 --key-threshold=1
Unseal Key 1: pk8vLxxC68h+nSM9whHhHvVhxpWwmzJyVWWhP1drZZA=

Initial Root Token: s.gnjE3RB13EfOnYqTFib7353J

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```
После распечатывания:
```
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.3.1
Cluster Name           vault-cluster-360d06ca
Cluster ID             f4b638b4-b251-d15d-4449-78fbb2fe22cc
HA Enabled             true
HA Cluster             https://10.8.2.11:8201
HA Mode                standby
Active Node Address    http://10.8.2.11:8200
```
Список бэкендов аутентификации:
```
/ $ vault auth list
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_ba7db502    token based credentials
```
После создания секретов:
```
/ $  vault read otus/otus-ro/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus
/ $ vault kv get otus/otus-rw/config
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
```
После включения авторизации через k8s
```
/ $ vault auth list
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_c319747c    n/a
token/         token         auth_token_ba7db502         token based credentials
```
Чтобы иметь возможность обновлять записи, необходима capabilities: update

Выдача сертификата:
```
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUU7IZHKvBap+Edi7SJWM//VF7PNQwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMDAyMjAwOTM1MDZaFw0yNTAy
MTgwOTM1MzZaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPOljmGXsTv/
6bYYxe7wYIwulXjtJ8TcFb2EGHZK7rCc/QLScGp1TADIq9vVwH0fME3Zc7agK5Ic
2D4VgGRwh39md+Q8FdQh5bqZ3uBA7K4E799Ltbjt8XWksAxnRlpIijsR0BjDikiN
0WFA//oX50UJgHLEvMLuRjoeaeW4FeII0EOb9l+5SObctVgCz0LgNRG504qtMw9z
79m9SJHh1HQu3AlCyR7FxGZDuhDEw4LgwQkZl4q5QhLUoTkLvcCKXM8oxMJ8rbBe
8kalnTe6rwFjCAcQLGdJ7abQFbtF5dudYrL0ZLYHXWEAIwBBKyS6TnkmOKqmfCkB
L4aki1PDt0ECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUpDbycAs311acxdlCu1HwiABPkXMwHwYDVR0jBBgwFoAU
sIyFAmKIZ4qPEoy3td6DL/nhNxAwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
k1XtgDBX/MfC+t2E+qcL2uE+MPWo0H5g8xYwcU2Jbz2jE5zyB4l+gd/BeUIC20qZ
xbaLJlOgoghkLUuRyDv1QMPNHBEDMlRf+P+lf4WiByTClwsWj8Tt/QCxt3b+DBNg
GdWCv9QLCgj/rP7tI3jkn326uB9qegr+sC8vBChycMTu/Ojh5jSDTZh1PRKtgv7M
zRz4NdFFCkCCUtsm1zZOyDq2Fbia5EJ8rulJSTyrMlhTLsZwB1u9iBbRuLQBf2vM
dt1QEKyNeYwXPVXDqqbf0YXIHR1ivUc6wZ1YrVSR4Gv0I5Ygnob1GqUQeUIBPdYT
jlbZSnCdLxq/LbpK4XCdxA==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDZzCCAk+gAwIBAgIUSnn+OKtCsforJZS+b9levMifZZ4wDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTIwMDIyMDEwMDY0OVoXDTIwMDIyMTEwMDcxOFowHDEaMBgGA1UEAxMRZ2l0
bGFiLmV4YW1wbGUucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC7
zS8aaCiimytmeUAbfxLI+td5dR+21nVVK32FI7KVLoR1ZJ7vI/jopVsMZD50BWXb
waT5FvqmbsFF2zllFIZHfCVNnOzXNAXqf4b+nxAIejR+L7cAAkQwYaBGSYcda0CH
hB29NKGkTLeXsj38Jen5c5wNynH0zQ31HxgqW7Qbt7IvBp/YaiEbr+IL9MSh3L4e
vnXv4Y/SXWMoXHHz26adSiR0ZeUalBXC+Pb9QcHNQ9mrF+p7synlJWzSNx8Q7/CY
CGjBC+sXUyT8HpncC/toXsdMiTKMJbvts6fYJMCfcMH7g3Haj9c05Rb+/nGZox61
FBqFgHNEJto94J1IewFxAgMBAAGjgZAwgY0wDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQUoW99JYGludUbfxfA
odpDqZZTSs4wHwYDVR0jBBgwFoAUpDbycAs311acxdlCu1HwiABPkXMwHAYDVR0R
BBUwE4IRZ2l0bGFiLmV4YW1wbGUucnUwDQYJKoZIhvcNAQELBQADggEBACcF/5nn
k+Fe8oigk1MBnaKIHOgMrNBuGiho0WRBIjpRBsrLNVfnNmQu8sB6TIhXyBAWHbm9
TYKrhPoa+k7YbQPz+k0XE7pxazDSx3c50fH+O6WC7a3WEUjjkLdvuQoHfkOD1ZHX
9g3SzfXP2iG9F/8Ax/sqFoOeGShEKGUg0QQyFhw/5HWkYTKdm2KwDOnceVv0i7Pw
7YSHjp7Yc/nJ9FGYB3suy8VZrnDOxYdYx8CqooU2cQcv1IVnTUli33LpO4mSXOjf
nTy/YF6cnhdT9xbCq0+faQdT0GLRBhnN0ozT74mmL5XNlRauACw3QJOYcRlpGJGq
mok1ydTAx+u7LsI=
-----END CERTIFICATE-----
expiration          1582279638
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUU7IZHKvBap+Edi7SJWM//VF7PNQwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMDAyMjAwOTM1MDZaFw0yNTAy
MTgwOTM1MzZaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPOljmGXsTv/
6bYYxe7wYIwulXjtJ8TcFb2EGHZK7rCc/QLScGp1TADIq9vVwH0fME3Zc7agK5Ic
2D4VgGRwh39md+Q8FdQh5bqZ3uBA7K4E799Ltbjt8XWksAxnRlpIijsR0BjDikiN
0WFA//oX50UJgHLEvMLuRjoeaeW4FeII0EOb9l+5SObctVgCz0LgNRG504qtMw9z
79m9SJHh1HQu3AlCyR7FxGZDuhDEw4LgwQkZl4q5QhLUoTkLvcCKXM8oxMJ8rbBe
8kalnTe6rwFjCAcQLGdJ7abQFbtF5dudYrL0ZLYHXWEAIwBBKyS6TnkmOKqmfCkB
L4aki1PDt0ECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUpDbycAs311acxdlCu1HwiABPkXMwHwYDVR0jBBgwFoAU
sIyFAmKIZ4qPEoy3td6DL/nhNxAwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
k1XtgDBX/MfC+t2E+qcL2uE+MPWo0H5g8xYwcU2Jbz2jE5zyB4l+gd/BeUIC20qZ
xbaLJlOgoghkLUuRyDv1QMPNHBEDMlRf+P+lf4WiByTClwsWj8Tt/QCxt3b+DBNg
GdWCv9QLCgj/rP7tI3jkn326uB9qegr+sC8vBChycMTu/Ojh5jSDTZh1PRKtgv7M
zRz4NdFFCkCCUtsm1zZOyDq2Fbia5EJ8rulJSTyrMlhTLsZwB1u9iBbRuLQBf2vM
dt1QEKyNeYwXPVXDqqbf0YXIHR1ivUc6wZ1YrVSR4Gv0I5Ygnob1GqUQeUIBPdYT
jlbZSnCdLxq/LbpK4XCdxA==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAu80vGmgoopsrZnlAG38SyPrXeXUfttZ1VSt9hSOylS6EdWSe
7yP46KVbDGQ+dAVl28Gk+Rb6pm7BRds5ZRSGR3wlTZzs1zQF6n+G/p8QCHo0fi+3
AAJEMGGgRkmHHWtAh4QdvTShpEy3l7I9/CXp+XOcDcpx9M0N9R8YKlu0G7eyLwaf
2GohG6/iC/TEody+Hr517+GP0l1jKFxx89umnUokdGXlGpQVwvj2/UHBzUPZqxfq
e7Mp5SVs0jcfEO/wmAhowQvrF1Mk/B6Z3Av7aF7HTIkyjCW77bOn2CTAn3DB+4Nx
2o/XNOUW/v5xmaMetRQahYBzRCbaPeCdSHsBcQIDAQABAoIBAEt8L6CnmR2yzQEF
X3Ut4HkUCiPxpeuxy7uOHYe0T4WTCv68kP2CMfwg6rXSXR/5Q3XPIeZVDC51eg4A
DdPJKif1iHDn0HK3oGEfHT2e5aziodLOjvnb71ibBPb7eumiQG+39NQmIYqOo4S/
yhZdjuwLQgBxDNjeyutpsibkcUJqJSqK+zbcM90baQMDjFvzPPwbrDb/0+UnRkMt
Qe/wnSzMkd4AOa53ayNSoGRG/GAlWed8QDCdNCsC1WJ2V92R5tQL910z0cO+EBzY
cLM+Yzo1k+ItEZq5E51zC5poKal//igY8PZX0fY1gNg0DvxPsdJdnb/U1UKb7NER
Gpc7D4ECgYEA5Z8a6CKo//SPBjFoco/e+h7kNlnLyEUkgJSInhB3L2jtjzb747sr
NMYtOZXZm748nIUXdtT7kkwHvqH1G9xbPMkDIfEUKeGHBKb0gRchWV4YOyOP4+Ik
YKH3rbGhroSnqGNby5lbsqf1nWPtGss7LYNHvicKUOtXHJadh0t8OmkCgYEA0WAz
z3uHJJmPFTUFLwfoVl0aZ9VA9YKAUgXT+OxADMts+yQCbjsIdSBZBw9dac3IOYtm
tgnUmn34/+gwfzZGLKqMvH1orBY26aEPJH4ZKOLAyJxDRA2/Xa5yUk0JpjwLhLB6
mD9yYD170RfIrGHYhJT9KkRGTzZMyEaLzTvRXckCgYAcDrzy8IlF/VQcpZzlor7U
QUIRghdseUZkj8HBzrFBkci1XzqYMR6ubCjKiIz2guBVH84mLxAuaCvqF1Aj/2EG
pGlFlHeqRmyBHQVzYKgqi1zanRXP+qoHRMNG7hWbhYoXAU0OK8cQpYVVnggy5fJt
NUVm7s5L5PXYAG9vQMIFQQKBgHf/tBIRUUY3wFq+NYdb99wvpiemgIF1VwgrkO6U
sKzklkRlwgLdUJ6YeI3kT3yJVV0tuSNSBQi6dFBvCgSO3a9R3DFXivs+DCDgjyYy
I0dclnMjpCXH30rY5Wqn/oTI2y0kXE8P5gSkmGchQ4EQ3yA1p9dmpAlYLK+IRy3M
P9WJAoGBAIlg9ced6BSQrnTne3NjYhBx1Jv/uthL4H2CHjpsVfD5RC8Kq5gtf3s4
YHXDM/Xgow39nPhaNYNueWTNXD0KPrcgAAOepu05Yn7lL0Ca05M+serJnHRB+Eh+
7vMDMDPLg3fjyI0lzLiBoCOPR8sZbhGXLC4Y4aFotLGbq75Drejq
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       4a:79:fe:38:ab:42:b1:fa:2b:25:94:be:6f:d9:5e:bc:c8:9f:65:9e
```
Включение TLS для Vault server:
 - сгенерировать приватный ключ с помощью openssl `openssl genrsa -out vault.key 4096`
 - создать конфигурацию для csr
 - создать csr `openssl req -config=vault.cnf -new -key vault.key -nodes -out vault.csr`
 - создать csr в k8s
```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: my-svc.my-namespace
spec:
  request: $(cat vault.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```
- подписать csr `kubectl certificate approve`
- получить подписанный сертификат `kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' \
    | base64 --decode > vault.crt`
- создать k8s секрет, содержащий ключ и сертификат: `kubectl create secret tls vault-certs --key=vault.key --cert=vault.crt`
- Изменить настройки vault:
  * `tls_disable = 0`
  * `tls_key_file = "/vault/certs/tls.key"`
  * `tls_cert_file = "/vault/certs/tls.crt"`  
- Добавить монтирование секрета в файлы для подов Vault и протокол доступа https, а также переменную:
 * VAULT_CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

Проверка из временного пода в неймспейсе dafault:
```
/ # curl -v --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt --header "X-Vault-Token:s.LoHQYkGrtKAhKHuOhZOdeFge" $VAULT_ADDR/v1/otus/otus-ro/config
*   Trying 10.12.14.254...
* TCP_NODELAY set
* Connected to vault (10.12.14.254) port 8200 (#0)
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  CApath: none
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use http/1.1
* Server certificate:
*  subject: C=RU; ST=Moscow; O=Perflabs; OU=Development; CN=localhost
*  start date: Feb 20 11:00:23 2020 GMT
*  expire date: Feb 18 11:00:23 2025 GMT
*  subjectAltName: host "vault" matched cert's "vault"
*  issuer: CN=222ead9d-8175-40eb-8c7b-80935319c8a0
*  SSL certificate verify ok.
> GET /v1/otus/otus-ro/config HTTP/1.1
> Host: vault:8200
> User-Agent: curl/7.61.1
> Accept: */*
> X-Vault-Token:s.LoHQYkGrtKAhKHuOhZOdeFge
> 
< HTTP/1.1 200 OK
< Cache-Control: no-store
< Content-Type: application/json
< Date: Thu, 20 Feb 2020 12:31:20 GMT
< Content-Length: 207
< 
{"request_id":"ecb0c604-0b43-5913-24fa-28daadc0f1a3","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"password":"asajkjkahs","username":"otus"},"wrap_info":null,"warnings":null,"auth":null}
* Connection #0 to host vault left intact
``` 
Скриншоты, показывающие обновление сертификатов:
 - `kubernetes-vault/nginx/cert1.png` [cert1.png](otus-kuber-2019-12/wildermesser_platform/blob/kubernetes-vault/kubernetes-vault/nginx/cert1.png)
 - `kubernetes-vault/nginx/cert2.png` [cert2.png](otus-kuber-2019-12/wildermesser_platform/blob/kubernetes-vault/kubernetes-vault/nginx/cert2.png)
