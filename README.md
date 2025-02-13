# BigData - инструкция по установке кластера hadoop

## Архитектура:

team-76-jn: 192.168.1.14 - jump node

team-76-nn:   192.168.1.15 - на этой виртуальной машине будут располагаться: name_node, secondary name_node, data_node_1

team-76-dn-00    192.168.1.16 - data_node_2

team-76-dn-01    192.168.1.17 - data_node_3

---

## 0. Подготовка серверов

Эти шаги инструкции выполняем от имени пользователя с правами sudo

### 0.1. Обновление системы  
На каждой виртуальной машине выполните обновление пакетов:
```bash
sudo apt update && sudo apt upgrade -y
```

### 0.2. Настройка файла /etc/hosts  
Чтобы ноды могли узнавать имена друг друга, на всех узлах (jump node, name node и DataNode) откройте файл `/etc/hosts` и добавьте следующие строки (убедитесь, что в этом файле отсутствуют избыточные записи типа “localhost”, которые могут мешать разрешению имен):

Для редактирования файла `/etc/hosts` воспользуйтесь командой
```bash
sudo vim /etc/hosts
```
Для перехода на другую виртуальную машину:
```bash
sss 192.168.1.15
```
ip отличаться в зависимости от виртуальной машины

для jump node файл `/etc/hosts` должен выглядеть вот так:
```
127.0.0.1       localhost
127.0.1.1       team-76-jn

192.168.1.15    team-76-nn
192.168.1.16    team-76-dn-00
192.168.1.17    team-76-dn-01

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
для name node:
```
127.0.0.1 localhost
127.0.1.1 team-76-nn
192.168.1.14    team-76-jn
192.168.1.16    team-76-dn-00
192.168.1.17    team-76-dn-01

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
для data node 0:
```
127.0.0.1 localhost
127.0.1.1 team-76-dn-00

192.168.1.14    team-76-jn
192.168.1.16    team-76-dn-00
192.168.1.17    team-76-dn-01

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
для data node 1:
```
127.0.0.1 localhost
127.0.1.1 team-76-dn-01
192.168.1.14    team-76-jn
192.168.1.15    team-76-nn
192.168.1.16    team-76-dn-00

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Проверьте, что при пинге по именам (например, `ping team-76-nn`) система корректно разрешает адреса.

---

## 1. Установка Java

Hadoop требует установленную Java. Рекомендуется использовать OpenJDK 8 или выше.

### 1.0 Проверьте или у Вас уже установлена Java:
```bash
java -version
```
Если уже установлена переходите к пункту 2

### 1.1. Установка JDK  
На каждом узле:
```bash
sudo apt install openjdk-8-jdk -y
```

### 1.2. Проверка установки  
Убедитесь, что Java установлена:
```bash
java -version
```

---

## 2. Создание пользователя для работы с кластером

Для повышения безопасности создадим пользователя без прав sudo. Имя пользователя **hadoopuser** с паролем **Ske8High!**.

На каждом узле (или по крайней мере на тех, где будут запускаться сервисы) выполните следующие команды:

```bash
# Создание пользователя hadoopuser (интерактивно, где нужно будет ввести дополнительные данные)
sudo adduser hadoopuser

# Если требуется задать пароль автоматически, можно выполнить:
echo "hadoopuser:Ske8High!" | sudo chpasswd
```

После создания переключитесь на нового пользователя:

```bash
su - hadoopuser
```

Все последующие шаги установки и настройки кластера будут выполняться от имени **hadoopuser** (при необходимости для выполнения системных операций можно использовать sudo).

---

## 3. Обеспечение SSH-взаимодействия

Для корректной работы кластера необходимо настроить SSH-доступ без пароля между всеми узлами.

### 3.0. Проверка установлен ли OpenSSH
```bash
systemctl status ssh
```
Если, OpenSSH установлен, переходите к шагу 2.2 

### 3.1. Установка OpenSSH  
На каждом узле установите SSH-сервер (Этот шаг придется делать от имени пользователя с правами sudo):
```bash
sudo apt install openssh-server -y
```

### 3.2. Генерация SSH-ключей  
На мастер-ноде (team-76-nn) выполните:
```bash
ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa
```
Это создаст пару ключей без пароля.

### 3.3. Распространение публичного ключа  
Скопируйте публичный ключ на все ноды (включая мастер и DataNode-ы). Для каждого узла выполните:
```bash
ssh-copy-id <имя_ноды>
```
Например:
```bash
ssh-copy-id team-76-nn
ssh-copy-id team-76-dn-00
ssh-copy-id team-76-dn-01
```
Проверьте, что теперь можно зайти с мастера на другие ноды по SSH без ввода пароля:
```bash
ssh team-76-dn-00
```

---

## 4. Установка Hadoop

### 4.1. Загрузка и распаковка Hadoop  
На всех узлах скачайте стабильную версию Hadoop (например, 3.3.1) с официального сайта:
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz
tar -xzf hadoop-3.4.1.tar.gz
sudo mv hadoop-3.4.1 /opt/hadoop
```

### 4.2. Настройка переменных окружения  
Добавьте следующие строки в файл `~/.bashrc` (или `~/.profile`):
```bash
export HADOOP_HOME=/opt/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64  # путь может отличаться, проверьте вашу установку
```
Примените изменения:
```bash
source ~/.bashrc
```

---

## 5. Конфигурация Hadoop

Все конфигурационные файлы располагаются в каталоге `$HADOOP_HOME/etc/hadoop`. Основные из них для HDFS – **core-site.xml** и **hdfs-site.xml**.

### 5.1. Файл core-site.xml  
Откройте файл `$HADOOP_HOME/etc/hadoop/core-site.xml` и добавьте (или измените) следующий блок:
```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://team-76-nn:9000</value>
  </property>
</configuration>
```
Это определяет адрес NameNode (используем имя, указанное в /etc/hosts).

### 5.2. Файл hdfs-site.xml  
Настройте директории для хранения метаданных и данных, а также коэффициент репликации:
```xml
<configuration>
  <!-- Директория для NameNode -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///opt/hadoop/dfs/name</value>
  </property>

  <!-- Директория для DataNode -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///opt/hadoop/dfs/data</value>
  </property>

  <!-- Задаем коэффициент репликации, равный числу DataNode (3) -->
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
</configuration>
```
*Примечание:* На узле team-76-nn будет запущен и DataNode, поэтому для него создается и рабочая директория.

### 5.3. Распространение конфигурации  
Скопируйте настроенные файлы `core-site.xml` и `hdfs-site.xml` с мастера (team-76-nn) на все DataNode (team-76-dn-0 и team-76-dn-1). Это можно сделать с помощью scp или rsync:
```bash
scp /opt/hadoop/etc/hadoop/core-site.xml team-76-dn-0:/opt/hadoop/etc/hadoop/
scp /opt/hadoop/etc/hadoop/hdfs-site.xml team-76-dn-0:/opt/hadoop/etc/hadoop/

scp /opt/hadoop/etc/hadoop/core-site.xml team-76-dn-1:/opt/hadoop/etc/hadoop/
scp /opt/hadoop/etc/hadoop/hdfs-site.xml team-76-dn-1:/opt/hadoop/etc/hadoop/
```

---

## 6. Форматирование NameNode

**Важно:** Форматирование NameNode выполняется только один раз на мастер-ноде (team-76-nn).

На team-76-nn выполните:
```bash
hdfs namenode -format
```
После успешного форматирования в каталоге для NameNode должны появиться соответствующие файлы.

---

## 7. Запуск кластера

### 7.1. Запуск HDFS  
Сначала запустите демоны HDFS на всех узлах. На мастере (team-76-nn) можно использовать скрипт:
```bash
start-dfs.sh
```
Скрипт `start-dfs.sh` автоматически запустит:
- На team-76-nn: **NameNode**, **Secondary NameNode** и **DataNode**;
- На team-76-dn-0 и team-76-dn-1: **DataNode**.

Если автоматический запуск по каким-то причинам не срабатывает, можно запускать демоны вручную:

- На мастере (team-76-nn):
  ```bash
  hadoop-daemon.sh start namenode
  hadoop-daemon.sh start secondarynamenode
  hadoop-daemon.sh start datanode
  ```
- На каждом DataNode (team-76-dn-0 и team-76-dn-1):
  ```bash
  hadoop-daemon.sh start datanode
  ```

### 7.2. Логи  
Логи демонов располагаются в каталоге `$HADOOP_HOME/logs`. Проверьте их на наличие критических ошибок:
```bash
tail -n 50 /opt/hadoop/logs/*namenode*.log
tail -n 50 /opt/hadoop/logs/*datanode*.log
```

---

## 8. Проверка работоспособности кластера

### 8.1. Веб-интерфейс NameNode  
Откройте в браузере адрес:
```
http://team-76-nn:50070
```
(либо по IP-адресу: http://192.168.1.15:50070).  
В интерфейсе проверьте, что:
- Нет деградировавших нод;
- Присутствуют три DataNode (учитывая, что на мастере тоже запущен DataNode).

### 8.2. Отсутствие критических ошибок  
Просмотрите логи всех демонов. Если ошибок нет, кластер можно считать целостным.

---

## 9. Дополнительные моменты

- **SSH и сеть:**  
  Убедитесь, что между всеми узлами нет блокировок (firewall, настройки сети), и каждый сервер может обращаться по имени/IP к другим узлам.

- **Jump Node (team-76-jn):**  
  Хотя на jump node не будут запущены сервисы кластера, он может использоваться для управления кластером. Можно настроить на нём проксирование (например, через nginx) для доступа к веб-интерфейсам кластера, если это требуется.

- **Безопасность:**  
  Рекомендуется ограничить внешнее (интернет) взаимодействие с кластером, оставив его внутренней системой, а доступ к веб-интерфейсам обеспечивать через jump node или посредством VPN.

- **Проверка окружения:**  
  После развертывания рекомендуется выполнить тестовые операции (например, загрузку/чтение файлов через HDFS) для окончательной проверки работоспособности кластера.

---

Эта инструкция позволит вам развернуть целостный кластер HDFS, где все демоны (NameNode, Secondary NameNode и три DataNode) будут запущены и взаимодействовать между собой без критических ошибок, что обеспечит успешное выполнение последующих практических заданий.
