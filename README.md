# Пример плагина для модификации шаблонов конфигурационного файла nginx для хоста (сайта)

В ISPmanager 6 форма редактирования настроек хоста `webdomain.edit` была заменена на форму `site.edit`. Из-за этого перестали работать некоторые плагины, которые используют запись и чтение из базы.

Этот пример модуля написан по реальной задаче - ограничить доступ к сайту по http(s) со всех ip-адресов за исключением корпоративного VPN на уровне настроек отдельного хоста.

Модуль написан после обращения в службу поддержки ispsystem и ответа от отдела разработки.

## Добавление чекбокса в форму

Согласно документации для добавления плагина необходимо разместить xml-файл с описанием плагина в директории `/usr/local/mgr5/etc/xml`.

Мы добавляем чекбокс с именем `vpnrestrict`, соответственно наш плагин будет называться `ispmgr_mod_vpnrestrict.xml`

`/usr/local/mgr5/etc/xml/ispmgr_mod_vpnrestrict.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<mgrdata>
</mgrdata>
```

Чтобы чекбокс отображался в новой форме `site.edit`, сохранял свое значение в БД и читал из нее, в xml-файле плагина необходимо описывать чекбокс дважды: для формы `webdomain.edit` и формы `site.edit`. Языковые значения также нужно прописывать дважды.

При этом в описании для формы `site.edit` перед именем чекбокса обязательно должен присутствовать префикс `site_`.

`/usr/local/mgr5/etc/xml/ispmgr_mod_vpnrestrict.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<mgrdata>
    <metadata name="site.edit" type="form">
        <form>
            <page name="mainsettings">
                <field name="site_vpnrestrict">
                    <input type="checkbox" name="site_vpnrestrict" />
                </field>
            </page>
        </form>
    </metadata>

    <metadata name="webdomain.edit" type="form">
        <form>
            <page name="mainsettings">
                <field name="vpnrestrict">
                    <input type="checkbox" name="vpnrestrict" />
                </field>
            </page>
        </form>
    </metadata>

    <lang name="ru">
        <messages name="webdomain.edit">
            <msg name="vpnrestrict">Ограничения по VPN</msg>
            <msg name="hint_vpnrestrict">Включение ограничения http-доступов по VPN</msg>
        </messages>
        <messages name="site.edit">
            <msg name="site_vpnrestrict">Ограничения по VPN</msg>
            <msg name="hint_site_vpnrestrict">Включение ограничения http-доступов по VPN</msg>
        </messages>
    </lang>
</mgrdata>
```

## Сохранение в БД

Так как нам необходимо запоминать значение чекбокса, нужно добавить поле в базу данных. Несмотря на то, что новая форма называется `site.edit`, работает она по прежнему с таблицей `webdomain`.

Согласно документации для добавления столбца в эту таблицу, необходимо в директории `/usr/local/mgr5/etc/sql` создать поддиректорию `webdomain.addon`, а в ней файл с именем нашего чекбокса. Имя файла указывается без расширения.

В файле описания поля можно указать тип, длину, значение по умолчанию, уровень прав пользователя панели на чтение и запись. Нас интересует только тип. У нас указано значение bool, но в IPSmanager значение флагов сохраняется как off/on, поэтому поле будет создано как VARCHAR(3).

`/usr/local/mgr5/etc/sql/webdomain.addon/vpnrestrict`
```text
type=bool
```

Этого достаточно, чтобы значение чекбокса начало сохраняться в БД.

## Обработчик

Основная задача плагина - менять конфигурационный файл nginx для хоста, поэтому нам необходим обработчик. Для добавления обработчика к плагину его нужно описать в xml-файле плагина:

`/usr/local/mgr5/etc/xml/ispmgr_mod_vpnrestrict.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<mgrdata>
    <handler name="vpnrestrict" type="xml">
         <event name="webdomain.edit" after="yes" />
    </handler>
    ...
</mgrdata>
```

Имя обработчика указано как `vpnrestrict`. Обратите внимание, что в описании обработчика указана "старая" функция ISPmanager (`webdomain.edit`) и имя чекбокса без префикса `site_`.

Сам файл обработчика будет располагаться в директории `/usr/local/mgr5/addon/`, тут нужно создать файл с именем обработчика - `vpnrestrict`. На файл обработчика нужно установить права 750.

Обработчик может быть написан на любом языке. Наш будет bash-скриптом.

В этом файле мы получаем XML, сгенерированный функцией, и добавляем в конец значение нашего чекбокса в новом элементе:

`/usr/local/mgr5/addon/vpnrestrict`
```shell
#!/bin/bash

cat | sed 's|</doc>$|<params><VPNRESTRICT>'$PARAM_vpnrestrict'</VPNRESTRICT></params></doc>|'
```

## Шаблон

Шаблоны конфигурационных файлов хоста находятся в файле `/usr/local/mgr5/etc/templates/nginx-vhosts.template` и `/usr/local/mgr5/etc/templates/nginx-vhosts-ssl.template` для http и https соответственно.

В нашем примере изменяется только `nginx-vhosts.template`, но изменения для https будут соответствующие.

Нам нужно по условию установленного чекбокса добавить две директивы в шаблон:

`/usr/local/mgr5/etc/templates/nginx-vhosts.template`
```text
server {
...
	{% if $VPNRESTRICT == on %}
	allow xxx.xxx.xxx.xxx;
	deny all;
	{% endif %}
...
}
```

## Заключение

После перезапуска панели и удаления кэша чекбокс должен появиться в форме в блоке "Основные настройки".

`rm -rfv /usr/local/mgr5/var/.xmlcache/ && /usr/local/mgr5/sbin/mgrctl -m ispmgr exit`

---
Таким образом с помощью флагов возможна кастомизация шаблонов под различные условия и нужды.
