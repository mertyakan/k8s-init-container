# k8s-init-container
kubernetes sidecar container ile php container arasındaki volume iceriginin paylasilmasi

# 
- Bu icerikte kubernetes uzerindeki php ve nginx podlari, nginx sidecar olacak sekilde konfigure edilmistir,
- Fakat uygulmanin php icerisindeki css dosyalarına ulasması gerekmektedir,
- Sidecar olarak deploy edilen podlar arasında ilgili alani volume olarak baglanmistir,
- Bu islem icin, init container kullanılmistir,

# 
Ozetle init container'in bu stratejideki gorevi php uzerindeki css dosyasının bulundugu dizini kendi uzerinde /tmp/X alanında yada sizin belirleyeceginiz herhangi bir alanda rsync ile dosyaları kopyalamasi, init container'a baglanan volume uzerine esitlemesi olarak tanımlayabiliriz. Sonrasında bu volume'u nginx uzerine baglayarak ilgili alandaki css dosyasına erismesini saglayacagiz.

Helm kullanarak olusturdugum chart uzerinde asagidaki dosyalarda degisiklikler yapilmistir,

1. helmchartfolder/values.yaml ##helm chart'inin values.yaml dosyasi
2. helmchartfolder/template/volumes/public.yaml ##bu dosya eklenecek
3. helmchartfolder/template/deployment.yaml ##initcontainer'in ve deployment'in yapılan degisiklikleri alabilmesi icin bu dosyada editleme yapilacak,
4. init container icinde calisacak script'i istediginiz alanda barındırabilirsiniz,

Klasor agaci asagidaki gibi; bu makalede anlatilan konu disindaki diger dosyalari gormezden geliniz,

```
├── app
│   ├── Chart.lock
│   ├── Chart.yaml
│   ├── charts
│   │   ├── common-1.10.4.tgz
│   │   ├── elasticsearch-7.16.3.tgz
│   │   └── mysql-8.8.23.tgz
│   ├── templates
│   │   ├── NOTES.txt
│   │   ├── _helpers.tpl
│   │   ├── configmaps
│   │   │   └── configmaps.yaml
│   │   ├── deployment.yaml
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── secret.yaml
│   │   ├── service.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── tests
│   │   │   └── test-connection.yaml
│   │   └── volumes
│   │       └── public.yaml
│   └── values.yaml
└── externalConfigs
    ├── envVars
    │   └── envVars
    ├── mysql
    │   └── mysql-values.yaml
    ├── nginx
    │   ├── default.conf
    │   ├── nginx.conf
    │   └── nginx.conf.sample
    └── scripts
        └── init_command.sh  
```
 
# 1. values.yaml 
```sh
persistence:
  enabled: true
  extraVolumeClaims:
    public:
      storageClass: px-sharedv4-sc
      accessMode: ReadWriteMany
      size: 2Gi
      
extraVolumes:
  - name: public
    persistentVolumeClaim:
      claimName: public      
      
sidecars:
  - name: nginx
    image: image/m2:latest
    volumeMounts:
    - mountPath: /var/www/html/public
      name: public      

initContainers:

  - name: sync-volume
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    securityContext:
      runAsUser: 0
    command: ["/bin/sh"]
    args:
    - -c
    - |
      {{- if .Values.externalConfigs.scripts.init_command_sh }}
      {{ tpl .Values.externalConfigs.scripts.init_command_sh . | nindent 4 }}
      {{- end }}

    volumeMounts:
      - name: public
        mountPath: /tmp/public
```


# 2. volumes/public.yaml
```sh
{{- if and .Values.persistence.enabled }}
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: public
  labels: {{- include "common.labels.standard" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  {{- if .Values.commonAnnotations }}
  annotations: {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.persistence.hostPath }}
  storageClassName: ""
  {{- else }}
  storageClassName: {{ .Values.persistence.extraVolumeClaims.public.storageClass }}
  {{- end }}

  accessModes:
    - {{ .Values.persistence.extraVolumeClaims.public.accessMode | quote }}
  resources:
    requests:
      storage: {{ .Values.persistence.extraVolumeClaims.public.size | quote }}
{{- end -}}
```

# 3. template/deployment.yaml
```sh
spec:
      {{- with .Values.initContainers }}
      initContainers:
      {{- include "common.tplvalues.render" ( dict "value" . "context" $ ) | nindent 8}}
      {{- end }}
```

# 4. init_command.sh
```sh
/bin/bash <<'EOF'
rsync -avc /var/www/html/public/ /tmp/public/
EOF
```


 
