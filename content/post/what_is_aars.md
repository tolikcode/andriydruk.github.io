+++
categories = ["Development"]
date = "2015-07-15T21:39:00+03:00"
description = ""
draft = false
image = "/img/material-001.jpg"
tags = ["androiddev"]
title = "Что такое .aar библиотеки?"

+++

Уже прошло больше года с анонса Android [New Build System](http://tools.android.com/tech-docs/new-build-system), работа над которой кипит и по сей день: в экспериментальной области все еще находится несколько увлекательных возможностей [Jack and Jill Tool Chain](http://tools.android.com/tech-docs/jackandjill) и [NDK Integration](http://tools.android.com/tech-docs/new-build-system/gradle-experimental#TOC-Ndk-Integration).

Но одной из самых ожидаемых возможностей новой системы сборки стала поддержка aar бибилотек - нового бинарного формата бибилотек, который в отличии от jar бибилотек может содержать android ресурсы (res, assets, native libraries, etc).  В это статье я хотел поделиться своим опытом работы с aar бибилиотеками.
<!--more-->

##  1. Что такое .aar бибилотеки?
[Официальная документация](http://tools.android.com/tech-docs/new-build-system/aar-format) говорит о том что aar  - это бинарный дистрибутив Android Library Project и прдеставляет собой zip файл с расширением .aar, котороый содержит:

* /AndroidManifest.xml (mandatory)
* /classes.jar (mandatory)		
* /res/ (mandatory)
* /R.txt (mandatory)
* /assets/ (optional)
* /libs/*.jar (optional)
* /jni/<abi>/*.so (optional)
* /proguard.txt (optional)
* /lint.jar (optional)
	
Разберем подробней. AndroidManifest.xml это оригинальный манифест с вашей библиотеки. Файл сlasses.jar содержит скомпилированные java классы библиотеки (они будут слиты с остальными во время компиляции основного проекта). Исходя из документации R.txt это выходной файл aapt (Android Asset Packaging Tool) с флагом --output-text-symbols, тоесть это часть вашего R.java который будет слит с остальными результатами работы aapt при сборке основного проекта. Далее идут директории assets, res, libs и jni, которые содержат оригинальные файлы из каталога src вашей бибилотеки (часть содержимого каталога res может быть удаленно после выполнения обфускации). Файл proguard.txt копируется из вашей root директории библиотеки для того чтобы передать правила обфускации в основной проект. При  работе с .aar стоит учитывать что если флаг minifyEnabled в build.gradle файле вашей библиотеки установлен в true, то при сборке она будет обфусцирована. Также aar бибилитека может содержать пользовательские lint правила в [lint.jar](https://groups.google.com/forum/#!msg/adt-dev/seWAK5r1fjI/0ed2rztjDbEJ), они будут распостраняться и на основной проект.
	
## 2. Как посторить .aar библиотеку?

Для того чтобы построить aar бибилотеку достаточно выполнить ./grdlew assemble в корне вашего проекта. Эта команда соберет все flavour и buildTypes. Но по причине ограничений gradle по умолчанию приложениями будет использоваться всегда defaultRelease сборка вашей aar бибилотеки. Если вы хотите поменять default конфигурацию, это можно сделать так
	
~~~gradle
android {
    defaultPublishConfig "debug"
} 
~~~
	
или так:
	
~~~gradle
android {
    defaultPublishConfig "flavor1Debug"
}
~~~
	
В первом случае в качестве default конфигурации используется debug build, во втором debug build варианта flavor1. Кроме того вы можете включить публикацию всех конфигураций:

~~~gradle
android {        
    publishNonDefault true
}
~~~
	
## 3. Как подключить .aar бибилиотеку?

Существует 4 способа подключения aar библиотек:

- Подключение в качестве модуля
- Плоский репозиторий (Flat directory)
- Локальный репозиторий (Local maven repo)
- Удаленный репозиторий (Remote maven repo)

### 3.1 Подключение модуля

Это пожалуй самый простой способ и с ним знакомо большинство Android разработчиков. Вы можете импортировать библиотеку как модуль в ваш проект и установить зависимость в своем основном проекте следующим образом:

~~~gradle
dependencies {
    compile project(':your_library_module_name')
}
~~~

Также вы можете установить [зависимость на определную конфигурацию вашей библотеки](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Referencing-a-Library)

~~~gradle
dependencies {
    flavor1Compile project(path: ':your_library_module_name', configuration: 'flavor1Release')
    flavor2Compile project(path: ':your_library_module_name', configuration: 'flavor2Release')
}
~~~

Такой подход позволяет вам легко вносить изменения в код вашей библиотеки и тут же получать aar при перекомпиляции. По большому счету вам даже не стоит задумывать о том что aar файл строится при сборке вашего проекта, в данном случае он выступает лишь в роли промежуточного звена сборки. Дальше мы расмотрим подходы подлючения собраных aar файлов.

### 3.1 Плоский репозиторий
	
Самый простой способ подключения aar бибилотек в собраном виде это простое копирование в директорию проекта. Для того чтобы установить зависимость на такой файл нужно включить директорию в список своих репозиториев в build.gradle

~~~gradle
repositories {
    flatDir {
        dirs 'path_to_folder_with_aar'
    }
}
~~~

После этого можно подключать библиотеку:

~~~gradle
dependencies {
   compile 'com.example:library:1.0.0@aar'
}
~~~

Версия бибилиотеки и namespace в данном случая ни на что не влияют, так как flatDir репозитории их просто игнорируют.

### 3.2 Локальный репозиторий

Для использования такого подхода вам нужен maven репозиторий разположений локально на вашем компьютере. Существует множество готовых gradle плагинов для публикации aar бибилиотек, но мы воспользуемся консольной устилитой mvn. Для начала следует проверить наличие установленого [maven](https://maven.apache.org/) в вашей системе.
	
~~~zsh
mvn --version
~~~

Если он отсутвует, его следует установить. Для публикации артефакта (в нашем случае .aar  библиотеки) в репозиторий используйте команду:
	
~~~zsh
mvn install:install-file -Dfile=<path_to_your_aar> -DgroupId=<domain> -DartifactId=<artifact_id> -Dversion=<version> -Dpackaging=aar
~~~

Например:
	
~~~zsh
mvn install:install-file -Dfile=mDNSShared-release.aar -DgroupId=com.dnssd -DartifactId=mDNSShared -Dversion=1.0.0 -Dpackaging=aar
~~~

На самом деле команда mvn install выполнит копирование вашего aar файла в директорию локального репозитория и сгенерирует соотвествующий pom.xml файл. Перейдем в ~/.m2/repository директорию на вашем компьютере. После выполнения команды install в этой директории должны появится поддиректории вашего домена, в случае моего приемера это 

~~~zsh
~/.m2/repository/com/dnssd/mDNSShared/
~~~

В это каталоге располгаются все версии вашего aar артефакта (под этим термином подразумевается любой бинарный файл помещеный в maven репозиторий). В моем случае это директория 1.0.0 в которой содержится 3 файла: _remote.repositories, mDNSShared-1.0.1.aar, mDNSShared-1.0.1.pom. Теперь подключим бибилотеку из локального репозитория. Прежде всего нужно добавить локальный maven репозиторий в список доступных:

~~~gradle
repositories {
    mavenLocal()
}
~~~
    
После этого можно устанавливать зависимости с помощью ключего слова compile также как мы это делает с flatDir

~~~gradle
dependencies {
    compile 'com.dnnsd:mDNSShared:1.0.0'
}
~~~

Именно такой подход распотсранения aar использован в Android Support Libraries. Все локальные maven репозитории разпаковуются и обновляются вместе с sdk.  Можете заглянуть в директории ANDROID_HOME/extras/android/m2repository или ANDROID_HOME/extras/google/m2repository.
	
### 3.3 Удаленный репозиторий

Подход с использвованием удаленного репозитория ничем не отличается от локального за исключением необходимости указания ссылки на репозиторий. Gradle соддержит ссылки на Maven Central и JCenter репозитории. Кроме этого gradle позволяет указывать прямой url на сервер maven репозитория:
	
~~~gradle
repositories {
    mavenCentral()
    jCenter()
    maven { url 'https://maven.fabric.io/repo' }
}
~~~

Установка зависимостей аналогична предыдущим способам:

~~~gradle
dependencies {
    compile('com.crashlytics.sdk.android:crashlytics:2.2.4@aar') {
        transitive = true;
    }
}
~~~
    
Вот и все.