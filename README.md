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
ssh 192.168.1.15
```
ip отличаться в зависимости от виртуальной машины

для `team-76-jn` файл `/etc/hosts` должен выглядеть вот так:
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
для `team-76-nn`:
```
127.0.0.1 localhost
192.168.1.15 team-76-nn
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
**Важно:** поменять `127.0.1.1 team-76-nn` на `192.168.1.15 team-76-nn`, иначе namenode не будет слушать datanode на других виртуалках
для `team-76-dn-00`:
```
127.0.0.1 localhost
127.0.1.1 team-76-dn-00

192.168.1.14    team-76-jn
192.168.1.15    team-76-nn
192.168.1.17    team-76-dn-01

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
для `team-76-dn-01`:
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
Если, OpenSSH установлен, переходите к шагу 3.2 

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
На всех узлах (team-76-nn, team-76-dn-00, team-76-dn-01) выполните следующие шаги:
### 4.1. Загрузка и распаковка Hadoop
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz
tar -xzf hadoop-3.4.1.tar.gz
```

### 4.2. Настройка переменных окружения HADOOP_HOME
Перейдите в созданную директорию hadoop-3.4.1 и скопируйте путь к ней:
```bash
cd hadoop-3.4.1
pwd
```
скорее всего путь будет выглядить вот так: **/home/hadoopuser/hadoop-3.4.1**, используйте его для добавления переменной окружения:
```bash
export HADOOP_HOME=/home/hadoopuser/hadoop-3.4.1
```

### 4.3. Настройка переменных окружения JAVA_HOME
```bash
which java
```
После того как выяснили какая у нас java уточняем путь до нее:
```bash
readlink -f /usr/bin/java
```
Копируем полученный путь до /bin. Путь будет выглядеть так: **/usr/lib/jvm/java-11-openjdk-amd64**

Перейдем в файл **hadoop-env.sh** 
```bash
vim $HADOOP_HOME/etc/hadoop/hadoop-env.sh
```
В нем расскоментируем строку JAVA_HOME и пропишем нужный путь:
```
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```
Выходим из фала и добавляем переменную окружения:
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```
### 4.4. Настройка переменных окружения PATH
Добавим папки исполняемые hadoop в переменную окружения path:
```bash
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
### 4.5 Обобщение
Для того чтобы преременные окружения не слетали каждый раз при перезаходе запишем их в файл
```bash
vim ~/.profile
```
Добавляем эти строчки внизу файла:
```
export HADOOP_HOME=/home/hadoopuser/hadoop-3.4.1
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
и перезапускаем командой:
```bash
source ~/.profile
```

Проверим что все установлено
```bash
hadoop version
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
Настройте директории для хранения метаданных и данных, а также коэффициент репликации.
Откройте файл `$HADOOP_HOME/etc/hadoop/hdfs-site.xml` и добавьте (или измените) следующий блок:
```xml
<configuration>
  <!-- Директория для NameNode -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///home/hadoopuser/hadoop-3.4.1/dfs/name</value>
  </property>

  <!-- Директория для DataNode -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:///home/hadoopuser/hadoop-3.4.1/dfs/data</value>
  </property>

  <!-- Задаем коэффициент репликации -->
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
</configuration>
```
*Примечание:* На узле team-76-nn будет запущен и DataNode, поэтому для него создается и рабочая директория.

### 5.3. Распространение конфигурации  
Скопируйте настроенные файлы `core-site.xml` и `hdfs-site.xml` с мастера (team-76-nn) на все DataNode (team-76-dn-00 и team-76-dn-01). Это можно сделать с помощью scp или rsync:
```bash
scp /home/hadoopuser/hadoop-3.4.1/etc/hadoop/core-site.xml hadoopuser@team-76-dn-00:/home/hadoopuser/hadoop-3.4.1/etc/hadoop/
scp /home/hadoopuser/hadoop-3.4.1/etc/hadoop/hdfs-site.xml hadoopuser@team-76-dn-00:/home/hadoopuser/hadoop-3.4.1/etc/hadoop/

scp /home/hadoopuser/hadoop-3.4.1/etc/hadoop/core-site.xml hadoopuser@team-76-dn-01:/home/hadoopuser/hadoop-3.4.1/etc/hadoop/
scp /home/hadoopuser/hadoop-3.4.1/etc/hadoop/hdfs-site.xml hadoopuser@team-76-dn-01:/home/hadoopuser/hadoop-3.4.1/etc/hadoop/
```
### 5.4. Настройка узлов datanode
Откройте файл `$HADOOP_HOME/etc/hadoop/workers` на namenode и запишите туда нужные узлы
```bash
team-76-nn
team-76-dn-00
team-76-dn-01
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
На мастере (team-76-nn) запустим скрипт:
```bash
start-dfs.sh
```
Скрипт `start-dfs.sh` автоматически запустит:
- На team-76-nn: **NameNode**, **Secondary NameNode** и **DataNode**;
- На team-76-dn-00 и team-76-dn-01: **DataNode**.

Если автоматический запуск по каким-то причинам не срабатывает, можно запускать демоны вручную:

- На мастере (team-76-nn):
  ```bash
  hadoop-daemon.sh start namenode
  hadoop-daemon.sh start secondarynamenode
  hadoop-daemon.sh start datanode
  ```
- На каждом DataNode (team-76-dn-00 и team-76-dn-01):
  ```bash
  hadoop-daemon.sh start datanode
  ```
  Чтобы остановить
  ```bash
  hadoop-daemon.sh stop datanode
  ```

### 7.2. Проверка запуска HDFS
Мы можем посмотреть запущенные демоны командой
```bash
jps
```
на namenode мы увидим:
```
86180 NameNode
87145 Jps
86362 DataNode
86575 SecondaryNameNode
```
на datanode'ах мы увидим:
```
87145 Jps
86362 DataNode
```

### 7.3. Логи  
Логи демонов располагаются в каталоге `$HADOOP_HOME/logs`. Проверьте их на наличие критических ошибок:
```bash
tail -n 50 hadoop-3.4.1/logs/<имя_ноды>.log
tail -n 50 hadoop-3.4.1/logs/hadoop-hadoopuser-namenode-team-76-nn.log
# или для просмотра в реальном времени:
tail -f hadoop-hadoopuser-datanode-team-76-dn-00.log
```

---

## 8. Проверка работоспособности кластера

### 8.1. Веб-интерфейс NameNode

На вашем компьютере запустите команду которая перенаправит ваш localhost:9870 на team-76-nn:9870
```bash
ssh -L 9870:team-76-nn:9870 -N -f hadoopuser@X.X.X.X
```
Откройте в браузере адрес:
```
http://localhost:9870
```
Аналогично для datanode:
```
ssh -L 9864:team-76-nn:9864 -N -f hadoopuser@X.X.X.X
```
Откройте в браузере адрес:
```
http://localhost:9864
```
  Если страница не открывается по указанному адресу проверьте, что вебинтерфейс запущен, для этого на team-76-nn запустите:
  ```bash
  curl -I http://localhost:9870
  ```
  если все правильно вы увидите `HTTP/1.1 302 Found`
  
  аналогично на team-76-jn проверьте, что сервис доступен:
  ```bash
  curl -I http://team-76-nn:9870
  ```
В интерфейсе проверьте, что:
- Нет деградировавших нод;
- Присутствуют три DataNode (учитывая, что на мастере тоже запущен DataNode).
