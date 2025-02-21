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

- Setando o disco:
```bash
qm set 150 --scsihw virtio-scsi-pci --scsi0 VOL-01:vm-150-disk-0
```

- Habilitando o Cloudinit:
```bash
qm set 150 --ide2 local-lvm:cloudinit
```

- Colocando o scsi0 como disco de boot:
```bash
qm set 150 --boot c --bootdisk scsi0
```

- Ajustando o monitor:
```bash
qm set 150 --serial0 socket --vga serial0
```

- Habilitar o agent:
```bash
qm set 150 --agent enabled=1
```

- Redimensionando o disco:
```bash
qm disk resize 150 scsi0 +30G
```

- Adicionando dependencias a serem instaladas:
```bash
qm set 150 --cicustom "vendor=local:snippets/vendor.yaml"
```

- Template:
```bash
qm template 150
```
