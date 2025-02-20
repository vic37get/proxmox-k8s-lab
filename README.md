# proxmox-k8s-lab

## Comandos de criação da VM Template:

- Primeiro comando:

```bash
apt install libguestfs-tools -y
```

- Comando para personalizar a imagem:

```bash
virt-customize --add jammy-server-cloudimg-amd64.img \
 --install qemu-guest-agent
```

- Comando para criar a máquina virtual:

```bash
qm create <NUMERO DA MÁQUINA> \
 --name <NOME DA MÁQUINA> \
 --numa 0 \
 --ostype l26 \
 --cpu cputype=host \
 --cores 3 \
 --sockets 2 \
 --memory 4096 \
 --net0 virtio,bridge=vmbr1,tag=11
```

- Comando para importar o disco:
```bash
qm importdisk 150 /tmp/jammy-server-cloudimg-amd64.img VOL-01
```
