+++
categories = ["Development", "golang"]
date = "2015-08-08T23:23:13+03:00"
description = ""
draft = true
image = "/img/about-bg.jpg"
tags = ["development"]
title = "Result Receiver"

+++


Есть одна неприятная особенность у ResultReciver: поскольку это IPC объект враппер IBinder, он не может быть сохраненым на диск в случае если система уничтожит вашу Activity. Причем система попытается подсунуть вам новый экземпляр ResultReciver, вместо экземпляра вашего наследника ResultReceiver. Следовательно при попытке считать его из Bundle вы получите ClassCastException. 


