---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date | time.Format "2006-01-02" }}
timezone: UTC+8
---

