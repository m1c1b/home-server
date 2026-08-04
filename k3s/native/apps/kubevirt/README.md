# Каталог установочных образов ВМ (KubeVirt)

ISO-образы лежат в обычной папке на диске ноды: **`/DATA/vm-images`**.
Крошечный nginx-pod раздаёт её по http, CDI импортирует ISO в PVC и публикует
как `DataSource`. Виртуалка подключает ISO как **cdrom** и грузится в установщик
— Ubuntu ставится руками через VNC на пустой диск.

```
/DATA/vm-images/ubuntu/ubuntu-24.04.4-desktop-amd64.iso   (просто файл на фс)
      │  nginx (image-fileserver.yml) раздаёт по http
      ▼  DataVolume import (image-catalog.yml)
ISO PVC ──> DataSource "ubuntu-noble-iso" (namespace golden-images)
      │  sourceRef (cdrom)
      ▼
VM ubuntu-vm (namespace vms) грузится в установщик
      │  ставит систему на пустой rootdisk (StorageClass vm-disks)
      ▼
/DATA/vm-disks/vms/ubuntu-vm-rootdisk/   (физический системный диск ВМ)
```

## Файлы

- `image-fileserver.yml` — namespace `golden-images` + nginx с hostPath на `/DATA/vm-images`.
- `image-catalog.yml` — DataVolume+DataSource для ISO (импорт через nginx).
- `storageclass-vm-disks.yml` — StorageClass `vm-disks`: диски ВМ на `/DATA` (не на root).
- `ubuntu-vm.yml` — VM: cdrom (ISO) + пустой rootdisk на `vm-disks`, грузится в установщик.

## Где лежит системный диск ВМ

Обычный `local-path` кладёт тома на root `/` (мало места). StorageClass `vm-disks`
через `nodePath` + `pathPattern` перенаправляет диски ВМ на большой `/DATA` с
предсказуемым путём:

```
/DATA/vm-disks/<namespace>/<pvc-name>/disk.img
```

Для ubuntu-vm системный диск физически будет здесь:

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
```

## Установка Ubuntu

**1. Применить StorageClass для дисков ВМ** (папка /DATA/vm-disks уже создана):

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/storageclass-vm-disks.yml
```

**2. Применить и запустить VM:**

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/ubuntu-vm.yml
virtctl --context home-k3s start ubuntu-vm -n vms
```

**3. Подключиться к установщику по VNC и поставить систему на диск:**

```bash
virtctl --context home-k3s vnc ubuntu-vm -n vms
```

Проходишь установщик Ubuntu Desktop как обычно, ставишь на диск (virtio, ~25Gi).

**4. После установки — убрать boot с cdrom**, иначе VM опять грузится в установщик.
В `ubuntu-vm.yml` в блоке `disks` удали cdrom-диск (и его volume), либо поменяй
bootOrder так, чтобы rootdisk был первым. Затем:

```bash
kubectl --context home-k3s apply -f k3s/native/apps/kubevirt/ubuntu-vm.yml
virtctl --context home-k3s restart ubuntu-vm -n vms
```

## Добавить другой ISO

1. Положить `.iso` на ноду: `ssh home 'curl -Lo /DATA/vm-images/<os>/<name>.iso <url>'`.
2. В `image-catalog.yml` добавить пару DataVolume+DataSource (скопировать блок,
   поменять имя/url/DataSource).
3. `kubectl apply -f image-catalog.yml`.

## Управление VM

```bash
virtctl --context home-k3s start   ubuntu-vm -n vms
virtctl --context home-k3s stop    ubuntu-vm -n vms
virtctl --context home-k3s restart ubuntu-vm -n vms
kubectl  --context home-k3s delete -f k3s/native/apps/kubevirt/ubuntu-vm.yml
```
