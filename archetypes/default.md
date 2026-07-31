+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
description = '{{ replace .File.ContentBaseName "-" " " | title }}'
+++
