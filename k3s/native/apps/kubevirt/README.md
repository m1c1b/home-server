# Каталог установочных образов ВМ (KubeVirt)

ISO-образы лежат в обычной папке на диске ноды: **`/DATA/vm-images`**.
Крошечный nginx-pod раздаёт её по http, CDI импортирует ISO в PVC и публикует
как `DataSource`. Виртуалка клонирует ISO как **cdrom** и грузится в установщик
— Ubuntu ставится руками через VNC на пустой диск.

```
/DATA/vm-images/ubuntu/ubuntu-24.04.4-desktop-amd64.iso   (просто файл на фс)
      │  nginx (image-fileserver.yml) раздаёт по http
      ▼  DataVolume import (image-catalog.yml)
ISO PVC ──> DataSource "ubuntu-noble-iso" (namespace golden-images)
      │  sourceRef (клон, cross-namespace -> нужен clone-rbac.yml)
      ▼
VM ubuntu-vm (namespace vms) грузится в установщик
      │  ставит систему на пустой rootdisk (StorageClass vm-disks)
      ▼
/DATA/vm-disks/vms/ubuntu-vm-rootdisk/disk.img   (физический системный диск ВМ)
```

## Файлы

- `image-fileserver.yml` — namespace `golden-images` + nginx с hostPath на `/DATA/vm-images`.
- `image-catalog.yml` — DataVolume+DataSource для ISO (импорт через nginx).
- `clone-rbac.yml` — RBAC: разрешает VM в `vms` клонировать образ из `golden-images`.
- `vm-storage-provisioner.yml` — отдельный local-path-provisioner + StorageClass `vm-disks` (диски ВМ на `/DATA`).
- `ubuntu-vm.yml` — VM: cdrom (ISO) + пустой rootdisk на `vm-disks`, грузится в установщик.

## Важные особенности этой связки (почему так)

Три нештатных момента, без которых оно не заводится на k3s + local-path:

1. **immediate-bind для импорта/клона.** `local-path` — не CSI, и при
   `WaitForFirstConsumer` CDI-импорт/клон впадает в дедлок (PVC ждёт потребителя,
   потребитель ждёт PVC). Лечится аннотацией на DataVolume:
   `cdi.kubevirt.io/storage.bind.immediate.requested: "true"`.
   Уже проставлена в `image-catalog.yml` (импорт ISO) и в `ubuntu-vm.yml`
   (dataVolumeTemplate ISO-клона).

2. **RBAC на cross-namespace clone.** VM в `vms` клонирует DataSource из
   `golden-images`. CDI требует, чтобы SA `vms:default` имел `create` на
   `datavolumes/source` в namespace-источнике. Даёт `clone-rbac.yml`.

3. **Отдельный provisioner для `/DATA`.** Системный k3s local-path кладёт тома на
   root `/` (27G, мало) и его configmap управляется k3s (правка перезатрётся при
   апгрейде). Поэтому диски ВМ обслуживает независимый provisioner из
   `vm-storage-provisioner.yml` (свой configmap с путём `/DATA/vm-disks`, своё имя
   `cluster.local/vm-disks`).

## Где лежит системный диск ВМ

StorageClass `vm-disks` (из `vm-storage-provisioner.yml`) через `pathPattern`
даёт предсказуемый путь на большом `/DATA`:

```
/DATA/vm-disks/<namespace>/<pvc-name>/disk.img
```

Для ubuntu-vm системный диск физически здесь:

```
/DATA/vm-disks/vms/ubuntu-vm-rootdisk/disk.img
```

Проверить после старта VM:

```bash
ssh home 'ls -la /DATA/vm-disks/vms/ubuntu-vm-rootdisk/'
```

Внутри VM это `/dev/vda` (virtio, 25Gi) — его и выбираешь в установщике.
ISO подключён как cdrom (`/dev/sr0`), не как диск.

## Первичная настройка

**1. Поднять файл-сервер:**

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/image-fileserver.yml
```

**2. Положить ISO в папку на ноде** (папка уже создана: `/DATA/vm-images/ubuntu`):

```bash
ssh home 'curl -Lo /DATA/vm-images/ubuntu/ubuntu-24.04.4-desktop-amd64.iso \
  https://releases.ubuntu.com/24.04/ubuntu-24.04.4-desktop-amd64.iso'
```

Проверить, что nginx отдаёт файл:
```bash
kubectl --context home-k3s -n golden-images run curl --rm -it --image=curlimages/curl --restart=Never -- \
  -sI http://vm-images-server.golden-images/ubuntu/ubuntu-24.04.4-desktop-amd64.iso
# должен быть HTTP/1.1 200 OK
```

**3. Применить каталог** (импортирует ISO в PVC):

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/image-catalog.yml
kubectl --context home-k3s -n golden-images get dv,datasource -w
# ждёшь: DataVolume -> Succeeded, DataSource READY=True
# (прогресс идёт в 2 этапа: скачивание с nginx + qemu-запись в PVC)
```

**4. Развернуть provisioner для дисков ВМ и RBAC клонирования:**

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/vm-storage-provisioner.yml
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/clone-rbac.yml
kubectl --context home-k3s -n vm-storage get pods   # provisioner должен быть Running
```

## Установка Ubuntu

**1. Применить и запустить VM:**

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/ubuntu-vm.yml
virtctl --context home-k3s start ubuntu-vm -n vms
```

Дождаться готовности (клон ISO + провижн rootdisk, пара минут):

```bash
kubectl --context home-k3s -n vms get vm,vmi,dv -w
# ждёшь: обе DV -> Succeeded, VMI -> Running, VM READY=True
```

**2. Подключиться к установщику по VNC и поставить систему на диск:**

```bash
virtctl --context home-k3s vnc ubuntu-vm -n vms
```

Проходишь установщик Ubuntu Desktop как обычно, ставишь на `/dev/vda` (virtio, ~25Gi).

**3. После установки — убрать boot с cdrom**, иначе VM опять грузится в установщик.
В `ubuntu-vm.yml` в блоке `disks` удали cdrom-диск (и его volume + dataVolumeTemplate),
либо поменяй bootOrder так, чтобы rootdisk был первым. Затем:

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/ubuntu-vm.yml
virtctl --context home-k3s restart ubuntu-vm -n vms
```

## Добавить другой ISO

1. Положить `.iso` на ноду: `ssh home 'curl -Lo /DATA/vm-images/<os>/<name>.iso <url>'`.
2. В `image-catalog.yml` добавить пару DataVolume+DataSource (скопировать блок,
   поменять имя/url/DataSource, сохранить annotation immediate-bind).
3. `kubectl apply -f image-catalog.yml`.

## Управление VM

```bash
virtctl --context home-k3s start   ubuntu-vm -n vms
virtctl --context home-k3s stop    ubuntu-vm -n vms
virtctl --context home-k3s restart ubuntu-vm -n vms
kubectl  --context home-k3s delete -f k3s/native/apps/kubevirt/ubuntu-vm.yml
```

## Диагностика (если VM висит в Provisioning)

```bash
# причина, если диски не создаются:
kubectl --context home-k3s -n vms get vm ubuntu-vm \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} ({.reason}: {.message}){"\n"}{end}'

# застрявший stateChangeRequest (мешает stop/start) — очистить:
kubectl --context home-k3s -n vms patch vm ubuntu-vm --type=merge \
  --subresource=status -p '{"status":{"stateChangeRequests":[]}}'

# логи provisioner дисков ВМ:
kubectl --context home-k3s -n vm-storage logs deploy/vm-disks-provisioner --tail=20
```
