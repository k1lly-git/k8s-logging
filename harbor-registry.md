## Установка Harbor

Перед установкой необходимо проверить, чтобы на хосте присутствовало следующее ПО с необходимыми версиями:
- Docker версии 17.06.0 и выше
- Docker Compose версии 1.18.0 и выше
- OpenSSL актуальной версии

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-online-installer-v2.11.0.tgz
tar -xzvf harbor-online-installer-v2.11.0.tgz
cd harbor
mv harbor.yml.tmpl harbor.yml
```

Генерация ключа и сертификата:
```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=yourdomain.com" \
 -key ca.key \
 -out ca.crt
```

В файле harbor.yml:
- В параметре hostname задаем IP-адрес или имя хоста, на котором будет находиться веб-интерфейс Harbor \
- В параметре certificate прописываем полный путь до файла с сертификатом с расширением .crt \
- В параметре private_key прописываем полный путь до файла с закрытым ключом с расширением .key \

```bash
sudo ./install.sh
```

Логин по умолчанию — admin, пароль — Harbor12345, можно создать нового пользователя в Users \
Создаем новый проект в разделе Projects -> New Project \
Добавляем нового пользователя в Members -> + USER \
Чтобы запушить образ, переходим в repositories -> Push Command \
- Устанавливаем пакет ca-certificates если он не установлен в системе:
```bash
sudo apt -y install ca-certificates
```
- Копируем корневой сертификат с расширением .crt в директорию /usr/local/share/ca-certificates:
```bash
sudo cp harbor.crt /usr/local/share/ca-certificates
```
- Устанавливаем сертификат:
```bash
sudo update-ca-certificates
```
Далее нужно залогиниться в docker через креды пользователя в Harbor
```bash
docker login <harbor-host.com>
```

Образ появится в веб-интерфейсе harbor откуда его можно запуллить
