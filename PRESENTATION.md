class: title, middle, center
<!-- css classes -->

# Ansible and Docker for Sysadmin

<!-- Notes -->
???
- Кратко о файлах и ансибле
- Рутинные операции 
- Массовые изменения
- Повторяемость
- Документирование
- Версионирование
- История изменений: перестали делаться бэкапы с роутера
- Galaxy
- Иммутабельность
- Идемпотентность
- Есть у меня два друга: Саня и Колян. Оба работают в аутсорсе - средненький такой аутсорс, по 10-20 компаний на обслуживании. Коля - ответственный, работящий парень. Может задержаться на работе, все делает до конца и правильно. Саша наоборот - лентяй, может на работе сериальчики посмотреть. И вообще - нелюбимый сотрудник у начальника. 

- При переходе с debian на RHEL Коля пошел в магазин. За веревкой и мылом
- 

---
## Кто я
----

<h3>
<table>
  <tr>
    <td><span style="font-size: 1em;"><i class="fas fa-user fa-fw"></i></span></td>
    <td>Дмитрий Бубнов</td>
    <td colspan="3" rowspan="7" bgcolor="white"><img src="img/MN-logo.png" alt="MN-logo" width="500"/></td>
  </tr>
  <tr>
    <td><i class="fas fa-home fa-fw"></i></td>
    <td>Челябинск, Бишкек</td>
  </tr>
  <tr>
    <td><i class="fas fa-graduation-cap fa-fw"></i></td>
    <td>Тренер Mikrotik</td>
  </tr>
  <tr>
    <td><i class="fas fa-award fa-fw"></i></td>
    <td>MTCNA, MTCRE, MTCTCE, MTCWE, MTCINE, CCNA R&S</td>
  </tr>
  <tr>
    <td><i class="fas fa-network-wired fa-fw"></i></td>
    <td>Сети</td>
  </tr>
  <tr>
    <td><i class="fas fa-chart-line fa-fw"></i></td>
    <td>Мониторинг</td>
  </tr>
  <tr>
    <td><i class="fas fa-magic fa-fw"></i></td>
    <td>Автоматизация</td>
  </tr>
</table>
</h3>

---
![Sysadminka](img/sysadminka.png)

---
## <i class="fas fa-table fa-fw"></i> О чем будем говорить
----

- ### Ansible
- ### Git
- ### Infrastructure as a Code
- ### Test-Driven Development
- ### Docker
- ### CI/CD
- ### Культура


---
## <img src="img/Ansible-logo-1.png" alt="Ansible-logo" width="70"/> Что такое Ansible
----
### Просто набор файлов!

```
basic-project
├── inventory
│ ├── group_vars
│ │ └── web.yml
│ ├── host_vars
│ │ └── db1.yml
│ └── hosts
└── site.yml
```
???
From [here](https://www.ansible.com/hubfs/2018_Content/AA%20BOS%202018%20Slides/Ansible%20Best%20Practices.pdf)

---
## <img src="img/Ansible-logo-1.png" alt="Ansible-logo" width="70"/> Или так
----
```
myapp
├── roles
│ ├── myapp
│ │ ├── tasks
│ │ │ └── main.yml
│ │ └── ...
│ ├── nginx
│ │ └── ...
│ └── proxy
│ └── ...
└── site.yml
```

---
## <img src="img/Ansible-logo-1.png" alt="Ansible-logo" width="70"/> Inventory
----
```
monitoring ansible_host=10.0.0.11 ansible_user=ansible ansible_port=222  ansible_sudo_pass="{{ vault_ansible_sudo_pass  }}"

[prometheus]
monitoring

[mikrotik-exporter]
monitoring

[grafana]
monitoring
```

---
## <img src="img/Ansible-logo-1.png" alt="Ansible-logo" width="70"/> Playbook
----
```
- name: Deploy prometheus
  hosts: prometheus
  roles:
    - ansible-prometheus
  tags:
    - prometheus
    - monitoring

- name: Deploy mikrotik-exporter
  hosts: mikrotik-exporter
  roles:
    - ansible-mikrotik-exporter
  tags:
    - monitoring
    - mikrotik-exporter    

- name: Deploy grafana
  hosts: grafana
  roles:
    - ansible-grafana
  tags:
    - grafana
    - monitoring
```

---
## <img src="img/Ansible-logo-1.png" alt="Ansible-logo" width="70"/> Запуск плэйбука
----
`ansible-playbook -i inventory.yml monitoring-playbook.yml`

или для отдельной роли

`ansible-playbook -i inventory.yml monitoring-playbook.yml --tags=prometheus`

---
## <img src="img/Ansible-logo-1.png" alt="Ansible-logo" width="70"/> Готовые плэйбуки
----
[cloudalchemy/ansible-prometheus](https://github.com/cloudalchemy/ansible-prometheus)

[geerlingguy/ansible-role-mysql](https://github.com/geerlingguy/ansible-role-mysql)



### Тысячи их

[Ansible Galaxy](https://galaxy.ansible.com/)

---
## <i class="fas fa-wrench fa-fw"></i> Нам срочно нужно 5 настроенных виртуалок
----

--
- ### С установленными обновлениями

--
- ### С юзерами, группами и ключами

--
- ### С настроенным фаерволом

--
- ### С установленной и первоначально настроенной MySQL

--
- ### Не забудь добавить их в мониторинг


---
## <i class="fas fa-wrench fa-fw"></i> Нам срочно нужно 5 настроенных виртуалок
----

```
- name: Postinstall configure
  hosts: all
  roles:
    - postinstall-config
  tags:
    - postinstall

- name: Deploy MySQL
  hosts: mysql
  roles:
    - ansible-role-mysql
  tags:
    - database
    - mysql
```
--
## Автоматизация рутинных действий

---
## <i class="fas fa-wrench fa-fw"></i> Нашли баг в Mikrotik, срочно закрой доступ к роутерам!
----

--
- ### 10 клиентов по 10 роутеров у каждого

--
- ### Часть может быть оффлайн

--
- ### При повторном запуске не должны создаваться дублирующие правила

---
## <i class="fas fa-wrench fa-fw"></i> Нашли баг в Mikrotik, срочно закрой доступ к роутерам!
----
```
  - name: add allowed addresses to list
    routeros_command:
      commands: "/ip firewall address-list add list=allow address={{ item }}"
    with_items:
      - "{{ routeros_list_allow }}"

  - name: drop input except allow list
    routeros_command:
      commands: "{{ item }}"
    with_items:
      - /ip firewall filter add chain=input src-address-list=allow action=accept
      - /ip firewall filter add chain=input action=drop
```

--
## Массовые изменения


---
## <i class="fas fa-wrench fa-fw"></i> Помнишь, мы настраивали IPSec три месяца назад? Надо сделать так же новому клиенту
----

--
- ### А на чем мы делали? Гугл показывает libreswan, strongswan, openswan, xl2tpd

--
- ### А как мы там роуты то прописали?

--
- ### Что-то не заводится сразу, видимо что-то забыли


---
## <i class="fas fa-wrench fa-fw"></i> Помнишь, мы настраивали IPSec три месяца назад? Надо сделать так же новому клиенту
----
```
- name: Magic playbook
  hosts: сидящие_в_зале
  roles: 
    - тут_должны
    - быть_роли
    - но_я_думаю
    - вы_и_так
    - все_поняли
```
--
## Повторяемость


---
## <i class="fas fa-wrench fa-fw"></i> У нас новый сотрудник. Покажи ему как у нас все работает
----

--
- ### 10 клиентов

--
- ### Естественно, все они разные

--
- ### Документация. Какая документация?


---
## <i class="fas fa-wrench fa-fw"></i> У нас новый сотрудник. Покажи ему как у нас все работает
----
cat ansible/group_vars/prometheus.yml
```
prometheus_storage_retention: "500d"
prometheus_storage_retention_size: "40GB"
prometheus_web_listen_address: "localhost:9090"

prometheus_alertmanager_config:
  - scheme: http
    static_configs:
      - targets:
        - "127.0.0.1:9093"

prometheus_scrape_configs:
  - job_name: "mikrotik"
    scrape_timeout: "60s"
    scrape_interval: "120s"
    metrics_path: "/metrics"
    static_configs:
      - targets:
        - "127.0.0.1:9436"
```

---
## <i class="fas fa-wrench fa-fw"></i> У нас новый сотрудник. Покажи ему как у нас все работает
----
### Документирование

---
## <i class="fas fa-wrench fa-fw"></i> Перестал работать бэкап
----

--
- ### Я что-то нажал и главный роутер у клиента сломался

--
- ### С 15 июля нет бэкапов с микротиков

--
- ### СРОЧНО ВОССТАНОВИ!!!!111один

---
## <i class="fas fa-wrench fa-fw"></i> Перестал работать бэкап
----
<img src="img/git-diff.png" alt="git-diff"/>

---
## <i class="fas fa-wrench fa-fw"></i> Перестал работать бэкап
----
### <img src="img/Git-Icon-Black.png" alt="git-logo" width="50"/> Git 
### Версионирование. История изменений

---
## <i class="fas fa-spinner fa-fw"></i> Так какие задачи решает Ansible?
----

### 1. Автоматизация рутинных действий
### 2. Массовые изменения
### 3. Повторяемость конфигураций
### 4. Документирование
### 5. Версионирование. История изменений


---
class: title, middle, center

# Test Driven Development

---
##  <i class="fas fa-vial fa-fw"></i> TDD
----

- ### Test Driven Development
- ### Неплохо было бы проверять плэйбуки перед их выпуском на боевую инфраструктуру
- ### Для этого есть специальные инструменты

---
## <i class="fas fa-vial fa-fw"></i> Разработка через тесты
----

<table>
  <tr>
    <td>Testinfra</td>
    <td colspan="2" rowspan="3"><img src="img/testinfra.png" alt="testinfra-logo" width="500"/></td>
  </tr>
  <tr>
    <td>Пишем тесты, проверяющие систему</td>
  </tr>
  <tr>
    <td>Подгоняем плэйбуки под выполнение этих тестов</td>
  </tr>
</table>

---
## <i class="fas fa-hat-wizard fa-fw"></i> Molecule
----

<table>
  <tr>
    <td>Первоначальная инициализация роли</td>
    <td colspan="2" rowspan="4"><img src="img/molecule.png" alt="molecule-logo" width="500"/></td>
  </tr>
  <tr>
    <td>Тесты на идемпотентность</td>
  </tr>
  <tr>
    <td>Верификация</td>
  </tr>
  <tr>
    <td>Driver/Provider</td>
  </tr>
</table>


---
## <i class="fas fa-hat-wizard fa-fw"></i> Molecule. Первоначальная инициализация роли
----

### В случае указания новой роли создает типовую структуру для роли ansible

---
## <i class="fas fa-hat-wizard fa-fw"></i> Molecule. Тесты на идемпотентность
----

###  При повторном запуске роли не должно быть произведено каких-либо изменений

---
## <i class="fas fa-hat-wizard fa-fw"></i> Molecule. Верификация
----

- ### Автоматический прогон тестов

<img src="img/testinfra.png" alt="testinfra-logo" width="500"/>


---
## <i class="fas fa-hat-wizard fa-fw"></i> Molecule. Driver/Provider
----

- ### Необязательно гонять тесты на заранее созданных виртуалках
- ### Vagrant
  - ### Libvirt
  - ### Parallels
  - ### VirtualBox (default)
  - ### VMware Fusion
- ### Docker

---
class: title, middle, center

## Docker

---
## <i class="fab fa-docker fa-fw"></i> Docker
----

- ### Позволяет запускать приложения в изолированном окружении
- ### Прекрасный инструмент для "пощупать как оно работает"
- ### Поматросил и бросил - контейнер не стоит ничего
- ### Легко мигрировать на другую машину

--
- ### ... или в другую серверную

--
- ### ... или в облако AWS, GCP, Azure, ...

---
class: title, middle, center

# Continuous Integration and Continuous Deployment

---
## <i class="fab fa-gitlab fa-fw"></i> CI/CD
----

- ### А зачем мне вся эта куча технологий, если и так всё работает?
- ### Но ведь уходит много времени на: 
  - ### написать плэйбук
  - ### написать тесты
  - ### запустить тесты
  - ### перенести конфиг на боевую инфраструктуру
  - ### настроить всю эту чепуху


---
## <i class="fab fa-gitlab fa-fw"></i> CI/CD
----

  - ### pipeline
  - ### Все делается автоматически:
    - ### git commit запускает тесты
    - ### при успешном прохождении тестов код разливается на prod
    - ### ваше время тратится только на единоразовую настройку системы
    - ### ... ну ОК, ещё на написание тестов и плэйбуков

---
class: title, middle, center

## Demo

Возможно, я уже выложил плэйбуки для деплоя мониторинга сети на  [GitHub](https://github.com/devi1/ansible)

---
## <i class="fas fa-info fa-fw"></i> Информация

#### Технологии

-   [Ansible Galaxy](https://galaxy.ansible.com)
-   [Ansible Community](https://www.ansible.com/community)
-   [Awesome Ansible List](https://github.com/geerlingguy/awesome-ansible)
-   [Molecule](https://habr.com/ru/post/437216/)
-   [Docker](https://www.digitalocean.com/community/tutorials/the-docker-ecosystem-an-overview-of-containerization)


#### Презентация
https://devi1.github.io/Pres_Ansible_docker/

#### Инструменты, которыми сделана эта презентация

-  [markdown](https://daringfireball.net/projects/markdown/)
-  [remark](https://github.com/gnab/remark)
-  [font-awesome](https://fontawesome.com/)
-  [Visual Studio Code](https://code.visualstudio.com/)
-  [hands](http://www.bubnovd.net)
---
## <i class="fas fa-hands-helping fa-fw"></i> Контакты
----

### <i class="fas fa-link fa-fw"></i> https://mikrotik-ninja.ru
### <i class="fab fa-telegram-plane fa-fw"></i> @mikrotikninja
### <i class="fab fa-telegram-plane fa-fw"></i> @sysadminka
### <i class="fab fa-telegram-plane fa-fw"></i> @bubnov
----

### <i class="fab fa-facebook-f fa-fw"></i><i class="fab fa-vk fa-fw"></i><i class="fab fa-twitter fa-fw"></i><i class="fab fa-linkedin-in fa-fw"></i><i class="fab fa-instagram fa-fw"></i><i class="fab fa-github fa-fw"></i>: 
### \- mikrotikninja
### \- sysadminka
###  \- bubnov

