---
author:
    name: Linode Community
    email: docs@linode.com
description: 'Deploy Graphite with Grafana on Ubuntu 14.04: Linode's Install & configure guide'
keywords: 'graphite,grafana,graphite-web,apache,python,postgreSQL,install,configure,monitoring tool'
license: '[CC BY-ND 3.0](http://creativecommons.org/licenses/by-nd/3.0/us/)'
modified: 'Tuesday, November 3rd, 2015'
modified_by:
  name: Sergey Pariev
published: 'Tuesday, November 3rd, 2015'
title: 'Deploy Graphite with Grafana on Ubuntu 14.04'
contributor:
  name: Sergey Pariev
  link: https://twitter.com/spariev
external_resources:
  - '[Installing Graphite](http://graphite.readthedocs.org/en/latest/install.html)'
  - '[Configuring Carbon](http://graphite.readthedocs.org/en/latest/config-carbon.html)'
  - '[Installing Grafana on Debian/Ubuntu](http://docs.grafana.org/installation/debian/)'
  - '[Adding Graphite data source to Grafana](http://docs.grafana.org/datasources/graphite/)'
---

[Graphite](http://graphite.readthedocs.org/en/latest/index.html) is an enterprise-scale monitoring tool that runs well on downmarket hardware. It stores numeric time-series data and renders graphs of this data on demand. This guide provides an introduction to the installation and configuration of Graphite together with [Grafana](http://grafana.org/) - a leading open-source application for visualizing large-scale measurement data - on Ubuntu 14.04 LTS.

{: .note}
>
>This guide is written for a non-root user. Commands that require elevated privileges are prefixed with `sudo`. If you're not familiar with the `sudo` command, you can check our [Users and Groups](/docs/tools-reference/linux-users-and-groups) guide.


## Prerequisites

1.  Follow the [Getting Started](/docs/getting-started) and [Securing Your Server](/docs/security/securing-your-server) guides, and [set the Linode's hostname](/docs/getting-started#setting-the-hostname).

    To check the hostname:

        hostname
        hostname -f

    The first command should show the short hostname, and the second command should show the fully qualified domain name (FQDN).

2.  Update the system:

        sudo apt-get update && sudo apt-get upgrade

{: .note}
>
>In this guide `example.com` will be used as a domain name, and `graphite` as the name of a non-root user. Substitute your own FQDN and username accordingly.


## Install Apache, Python tools and Graphite

1.  Install the system packages required for installing and using Graphite:

        sudo apt-get install build-essential python-dev apache2 libapache2-mod-wsgi libpq-dev python-psycopg2

2.  Install system packages for Graphite:

        sudo apt-get install graphite-web graphite-carbon

	The installer will ask if you want to remove Carbon database files on uninstall. It's better to answer 'No' to this question; you can always remove the files later. (They are located in `/var/lib/graphite/whisper`.)

## Configure Carbon

1.  Configure storage settings for test metrics: using `sudo` and your favorite text editor, add the following lines to file `/etc/carbon/storage-schemas.conf` after `[carbon]` section but before `[default_1min_for_1day]` section:

{: .file-excerpt}
/etc/carbon/storage-schemas.conf
:   ~~~ conf
	[test]
	pattern = ^test\.
	retentions = 5s:3h,1m:1d
    ~~~

	This file describes retention policies that Graphite will use for configured metrics. `pattern` is the regular expression to match against a metric name, and `retentions` is a comma-separated list of `frequency`:`history` value pairs.

	The section `[test]` you've just added will match all metrics starting with `test` and will save data every 5 seconds for 3 continuous hours and every 1 minute over the course of 1 day.

{: .note}
>
>`[test]` section is for testing purposes only and can be safely skipped.

	Also, the `[carbon]` section describes policies for internal Carbon metrics, while the `[default_1min_for_1day]` section contains default settings, which will be used if none of the previous patterns are matched.

	For more information on how to configure Carbon storage see [relevant section of Graphite documentation](http://graphite.readthedocs.org/en/latest/config-carbon.html#storage-schemas-conf).

2.  Configure aggregation settings by copying the defaut aggregation configuration to `/etc/carbon`:

		sudo cp /usr/share/doc/graphite-carbon/examples/storage-aggregation.conf.example /etc/carbon/storage-aggregation.conf

	`storage-aggregation.conf` describes aggregation policies Carbon uses to produce less detailed metrics, such as `1m:1d` retention in `[test]` section added in the previous step. By default, only the average of metric values is taken, which will result in data loss, if, for example, maximum and minimum values are needed. For this reason, `[min]`,`[max]` and `[sum]` sections are added in the configuration file.

	For more information on how to configure Carbon aggregation see [relevant section of Graphite documentation](http://graphite.readthedocs.org/en/latest/config-carbon.html#storage-aggregation-conf).

3.  Enable carbon-cache to run on boot: edit file `/etc/default/graphite-carbon` and change the value for `CARBON_CACHE_ENABLED` to `true`:

{: .file-excerpt}
/etc/default/graphite-carbon
:   ~~~ conf
	CARBON_CACHE_ENABLED=true
    ~~~

4. Start carbon-cache service:

		sudo service carbon-cache start


## Install PostgreSQL and prepare databases for graphite-web and Grafana

1.  Install PostgreSQL database for graphite-web application using [PostgreSQL installation guide](docs/databases/postgresql/ubuntu-12-04-precise-pangolin).

2.  As `postgres` user create database user for graphite-web app:

		createuser graphite --pwprompt

     You will be asked to provide a password for the new user. Once you have, create databases `graphite` and `grafana` with `graphite` user as owner:

		createdb -O graphite graphite
		createdb -O graphite grafana


## Set up graphite-web application

1.  Open graphite-web configuration file `/etc/graphite/local_settings.py` with `sudo` and make the changes described below.

2.  Find the `DATABASES` dictionary definition and update it with settings for PostgreSQL database created earlier:

{: .file-excerpt}
/etc/graphite/local_settings.py
:   ~~~ py
	DATABASES = {
		'default': {
			'NAME': 'graphite',
			'ENGINE': 'django.db.backends.postgresql_psycopg2',
			'USER': 'graphite',
			'PASSWORD': 'graphiteuserpassword',
			'HOST': '127.0.0.1',
			'PORT': ''
		}
	}
    ~~~

3.  Add the following lines to the end of the file:

{: .file-excerpt}
/etc/graphite/local_settings.py
:   ~~~ py
	USE_REMOTE_USER_AUTHENTICATION = True
	TIME_ZONE = 'Your/Timezone'
	SECRET_KEY = 'somelonganduniquesecretstring'
    ~~~

where TIME_ZONE is your time zone, which will be used in graphs (for possible values see TZ column value in [this timezones list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)), and SECRET_KEY is a long and unique string used as salt when hashing passwords.

4.  Save your changes.

5.  Initialize graphite-web database:

		sudo graphite-manage syncdb

	You will be asked to create a superuser account that will access the graphite-web interface.


## Configure Apache for graphite-web application

1.  Copy the graphite-web apache config file into apache `sites-available` directory:

		sudo cp /usr/share/graphite-web/apache2-graphite.conf /etc/apache2/sites-available

2.  Change the graphite-web port from 80 to 8080 (port 80 will be used for Grafana later). To do this, open file `/etc/apache2/sites-available/apache2-graphite.conf` and change line

		<VirtualHost *:80>
to

		<VirtualHost *:8080>

3.  To verify that Apache listens on port 8080, edit file `/etc/apache2/ports.conf` and add line `Listen 8080` after `Listen 80`:

{: .file-excerpt}
/etc/apache2/ports.conf
:   ~~~ conf
	Listen 80
	Listen 8080
    ~~~

4.  Disable default apache site to avoid conflicts:

		sudo a2dissite 000-default

5.  Enable graphite-web virtual site:

		sudo a2ensite apache2-graphite

6.  Apply changes:

		sudo service apache2 reload

Now you should be able to access `http://<your_server_name_or_ip>:8080` in the browser and see the Graphite home page.


## Feed Graphite sample data

1.  Send some test metrics by executing the following command:

		for i in 4 6 8 16 2; do echo "test.count $i `date +%s`" | nc -q0 127.0.0.1 2003; sleep 6; done

	Now, refresh the Graphite home page and you should see a new `test.count` metric in the tree on the left:

![Graphite test metric](/docs/assets/graphite_test_metric.png)

## Install and configure Grafana

1.  Using `sudo`, edit file `/etc/apt/sources.list` and add the following line:

{: .file-excerpt}
/etc/apt/sources.list
:   ~~~ conf
	deb https://packagecloud.io/grafana/stable/debian/ wheezy main
    ~~~

2.  Add the [Package Cloud](https://packagecloud.io/grafana) key to install signed packages.

		curl https://packagecloud.io/gpg.key | sudo apt-key add -

3.  Update apt settings and install Grafana:

		sudo apt-get update
		sudo apt-get install grafana

4.  Now, configure Grafana to use the PostgreSQL database created earlier. Using `sudo`, edit file `/etc/grafana/grafana.ini` and fill in the proper database configuration in `[database]` section:

{: .file-excerpt}
/etc/grafana/grafana.ini
:   ~~~ conf
	[database]
    # Either "mysql", "postgres" or "sqlite3", it's your choice
	type = postgres
	host = 127.0.0.1:5432
	name = grafana
	user = graphite
	password = graphiteuserpassword
    ~~~

5.  Next, configure domain and root_url in `/etc/grafana/grafana.ini` and set a more secure admin password and secret key:

{: .file-excerpt}
/etc/grafana/grafana.ini
:   ~~~ conf
	[server]
    protocol = http
	http_addr = 127.0.0.1
	http_port = 3000
	domain = example.com
	enforce_domain = true
	root_url = %(protocol)s://%(domain)s/

	[security]
	admin_user = admin
	admin_password = SecureAdminPass
	secret_key = somelongrandomstringkey
    ~~~

6.  Run proxy modules to enable Apache reverse proxying:

		sudo a2enmod proxy
		sudo a2enmod proxy_http
		sudo a2enmod xml2enc

7.  To proxy requests to Grafana, create an Apache site configuration file `/etc/apache2/sites-available/apache2-grafana.conf` with the following content (remember to change `example.com` to your own domain name):

{: .file}
/etc/apache2/sites-available/apache2-grafana.conf
:   ~~~ conf
	<VirtualHost *:80>
	 ProxyPreserveHost On
	 ProxyPass / http://127.0.0.1:3000/
	 ProxyPassReverse / http://127.0.0.1:3000/
	 ServerName example.com
	</VirtualHost>
	~~~

7.  Enable Grafana site configuration:

		sudo a2ensite apache2-grafana

8.  Configure Grafana to run at boot and start service:

		sudo update-rc.d grafana-server defaults 95 10
		sudo service grafana-server start

9.  Restart Apache2 to adopt new modules and configuration changes:

		sudo service apache2 restart

	At this point, you should be able to open `http://<your_domain>/` in a browser and see the Grafana login page.

## Add Graphite data source to Grafana

1.  Log in to Grafana using the `admin` credentials you specified in the configuration file.

2.  Click `Data Sources` and select `Add new`. Fill in all the fields as shown in the picture below:

![Add Data Source dialog](/docs/assets/graphite_grafana_data_source.png)

Click `Add` to create the new Data Source.

3.  Now, before creating a graph, add more test data for `test.count` metric:

		for i in 4 6 8 16 2; do echo "test.count $i `date +%s`" | nc -q0 127.0.0.1 2003; sleep 6; done

4.  Create a new dashboard by clicking the `Home` button and then `+ New`:

![Create new dashboard](/docs/assets/graphite_grafana_new_dashboard.png)

5.  Add a graph panel to the newly created dashboard:

![Create new graph panel](/docs/assets/graphite_grafana_new_graph.png)

6.  Next, edit graph panel properties - click `no title (click here)` and then click `Edit`:

![Edit graph panel](/docs/assets/graphite_grafana_edit_graph.png)

7.  With the graph panel properties dialog open, perform the following steps:

  A. Make sure the `graphite` data source you've created is chosen in combobox at the bottom right (marked 1 in the picture below).
  B. Click time period at the top right corner (marked as 2) and choose `Last 15 minutes`.
  C. Click on `select metric` and choose `test` and then `count` (marked as 3) to add the test metric you've added data for previously. At this point, visualization of the sample data should appear on the graph.
  D. Finally, click `Save` (marked 4 at the picture) to save the dashboard you've created.

[![Add test metric to the panel](/docs/assets/graphite_grafana_edit_graph_add_metric_small.png)](/docs/assets/graphite_grafana_edit_graph_add_metric.png)



