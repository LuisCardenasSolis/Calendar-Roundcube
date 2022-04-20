
# Calendar-Roundcube
Install Calendar Roundcube Webmail

### REFERENCES
- [CALDAV](https://github.com/JodliDev/calendar)
- [COMPOSER](https://getcomposer.org/download/)


## INSTALL PLUGIN
~~~
cd /pathTo/roundcubemail

composer config repositories.calendar vcs https://github.com/JodliDev/calendar
composer config repositories.libcalendaring vcs https://github.com/JodliDev/libcalendaring
composer config minimum-stability dev
composer require kolab/calendar (create and use token from any github account)
~~~

### REPLACE PLUGINS
~~~
cd plugins

rm -rf calendar libkolab libcalendaring (if exist)
git clone https://github.com/JodliDev/calendar.git
git clone https://github.com/kolab-roundcube-plugins-mirror/libkolab.git
git clone https://github.com/JodliDev/libcalendaring.git
~~~

### IMPORT DATABASE
~~~
cd /pathTo/roundcubemail
bin/initdb.sh --dir=plugins/calendar/drivers/database/SQL
~~~

### ENABLE CALENDAR PLUGIN
~~~
cd /pathTo/roundcubemail
vim config/config.inc.php
~~~
>`$config['plugins'] = [...,'calendar']`

___
## PLUGIN CONFLICT 

### UPDATE CARDDAV
En caso de utilizar el plugin carddav para la sincronización de contactos, este tiene que estar actualizado para que no entre en conflicto con el plugin calendar

[PLUGIN CARDDAV](https://github.com/mstilkerich/rcmcarddav)
*¡Cierra sesión en Roundcube! ¡Esto es importante porque RCMCardDAV ejecuta su procedimiento de inicialización/actualización de la base de datos solo cuando un usuario inicia sesión!*

~~~
cd /root
wget https://github.com/mstilkerich/rcmcarddav/releases/download/v4.3.0/carddav-v4.3.0.tar.gz
tar zxf carddav-v4.3.0.tar.gz
cd /pathTo/roundcubemail
cd plugins
mv carddav/ carddav.bkp
mv /root/carddav .

cp carddav/config.inc.php.dist carddav/config.inc.php
~~~
---
#### Edit Conf file CARDDAV
>cat carddav.bkp/config.inc.php | grep url  ***(ULR FROM PAST PLUGIN)***
> vim carddav/config.inc.php

~~~
<?php

//// ** GLOBAL SETTINGS
$prefs['_GLOBAL']['hide_preferences'] = true;
$prefs['_GLOBAL']['loglevel'] = \Psr\Log\LogLevel::WARNING;
$prefs['_GLOBAL']['loglevel_http'] = \Psr\Log\LogLevel::ERROR;

//// ** ADDRESSBOOK PRESETS
// Each addressbook preset takes the following form:

$prefs['ownCloud'] = [
    // required attributes
    'name'         =>  'ownCloud',
    'url'          =>  'ULR FROM PAST PLUGIN',

    // required attributes unless passwordless authentication is used (Kerberos)
    'username'     =>  '%u',
    'password'     =>  '%p',

    // optional attributes
    'active'       =>  true,
    'readonly'     =>  false,
    'refresh_time' => '02:00:00',
    //'rediscover_mode' => 'none',

    // attributes that are fixed (i.e., not editable by the user) and auto-updated for this preset
    'fixed'        =>  ['username','password'],
    'hide'        =>  false,
];
~~~

Restart Webserver
~~~
systemctl restart (nginx|httpd|apache2)
~~~
___
## FIX ERROR WHEN IMPORTING INVITATION FROM GMAIL

- Actualizar la zona horaria del PHP según su versión instalada:
	~~~
	vim /etc/php/php.ini
	~~~

	> date.timezone=America/Lima
	---

- Reiniciar el gestor de procesos de su version de PHP en caso este instalado:
	~~~
	systemctl restart php-fpm
	~~~
	---


- Con la zona horaria correcta simplemente corregir el código del archivo database_driver.php cambiando todas las coincidencias de la linea:

	~~~
	$this->rc->db->now(); 
	~~~
	change for ⇒ 
	~~~
	$now = new DateTime(); 
	$now = "'".$now ->format('Y-m-d H:i:s')."'";
	~~~
	
	Finalmente editar todas las coincidencias indicadas en el archivo indicado para poder solucionar el problema, solo para la linea 313 agregar todo lo siguiente:
	~~~
	vim plugins/calendar/drivers/database/database_driver.php
	~~~
	~~~
	private function _insert_event(&$event)
	    {
	        $event = $this->_save_preprocess($event);
	     //   $now   = $this->rc->db->now();
	        $now = new DateTime();
	        $now = "'".$now ->format('Y-m-d H:i:s')."'";

	        $tz_event = new DateTimeZone(date_default_timezone_get());
	        $start_event = new DateTime("@{$event['start']->getTimeStamp()}");
	        $end_event = new DateTime("@{$event['end']->getTimeStamp()}");
	        $start_event = $start_event->setTimezone($tz_event);
	        $end_event = $end_event->setTimezone($tz_event);

	        $this->rc->db->query(
	            "INSERT INTO `{$this->db_events}`"
	            . " (`calendar_id`, `created`, `changed`, `uid`, `recurrence_id`, `instance`,"
	                . " `isexception`, `start`, `end`, `all_day`, `recurrence`, `title`, `description`,"
	                . " `location`, `categories`, `url`, `free_busy`, `priority`, `sensitivity`,"
	                . " `status`, `attendees`, `alarms`, `notifyat`)"
	            . " VALUES (?, $now, $now, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
	            $event['calendar'],
	            strval($event['uid']),
	            isset($event['recurrence_id']) ? intval($event['recurrence_id']) : 0,
	            isset($event['_instance']) ? strval($event['_instance']) : '',
	            isset($event['isexception']) ? intval($event['isexception']) : 0,
	            //$event['start']->format(self::DB_DATE_FORMAT),
	            $start_event,
	            //$event['end']->format(self::DB_DATE_FORMAT),
	            $end_event,
	            intval($event['all_day']),
	            $event['_recurrence'],
	~~~
	
	Para parchar todo el archivo directamente simplemente reemplazar el driver por el archivo ya corregido de este repositorio:
	
	~~~
	cd /pathTo/roundcubemail
	cd plugins/calendar/drivers/database/

	mv database_driver.php database_driver.php.old

	wget https://raw.githubusercontent.com/LuisCardenasSolis/Calendar-Roundcube/main/database_driver.php
	~~~
