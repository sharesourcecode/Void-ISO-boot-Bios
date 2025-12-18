## Copilot

Claro, aqui está o guia **completo e corrigido** para configurar a sua imagem `.img` com o **SYSLINUX**, utilizando `LABEL=easy2` nas configurações do boot. Este guia foi ajustado com base nos problemas encontrados anteriormente e já incorpora as últimas correções para garantir que tudo funcione corretamente.

---

## **Processo Completo para Customizar a Imagem**

1. **Trabalhar com uma cópia da imagem original**
   Sempre crie uma cópia da imagem original para evitar alterações permanentes:
   ```bash
   cp easy-6.6.10-amd64.img easy-6.6.10-amd64-custom.img
   ```
   A partir de agora, todas as modificações serão feitas na cópia (`easy-6.6.10-amd64-custom.img`).

2. **Associar a imagem a um dispositivo loop**
   Associe a imagem ao dispositivo `loop` com mapeamento automático das partições:
   ```bash
   sudo losetup -Pf easy-6.6.10-amd64-custom.img
   ```

   Confirme que as partições foram devidamente mapeadas:
   ```bash
   lsblk | grep loop
   ```
   Saída esperada:
   ```plaintext
   loop0       7:0    0   953M  0 loop 
   ├─loop0p1 259:0    0     7M  0 part
   └─loop0p2 259:1    0   944M  0 part
   ```

   Se as partições não forem listadas, use:
   ```bash
   sudo partx -a /dev/loop0
   ```

3. **Monte a partição EFI**
   Esta é a partição onde configuraremos o SYSLINUX.

   - Crie um diretório para montagem:
     ```bash
     mkdir -p ~/easyimg_efi
     ```
   - Monte a partição EFI (`loop0p1`):
     ```bash
     sudo mount /dev/loop0p1 ~/easyimg_efi
     ```
   - Confirme o conteúdo montado:
     ```bash
     ls ~/easyimg_efi
     ```
     Deve listar arquivos como: `EFI`, `syslinux.cfg`, etc.

4. **Remova arquivos relacionados a outros bootloaders (caso existam)**
   Caso arquivos como `limine-bios.sys` ou `limine.cfg` estejam presentes, remova-os:
   ```bash
   sudo rm -f ~/easyimg_efi/limine-bios.sys
   sudo rm -f ~/easyimg_efi/limine.cfg
   ```

5. **Instale o SYSLINUX**
   Adicione os arquivos necessários do SYSLINUX à partição EFI:
   - Copie o arquivo `ldlinux.c32`:
     ```bash
     sudo cp /usr/lib/syslinux/ldlinux.c32 ~/easyimg_efi/
     ```
   - Copie o arquivo `ldlinux.e64` (para EFI de 64 bits):
     ```bash
     sudo cp /usr/lib/syslinux/efi64/ldlinux.e64 ~/easyimg_efi/
     ```
   - Confirme os arquivos copiados:
     ```bash
     ls ~/easyimg_efi
     ```
     Deve listar: `EFI`, `ldlinux.c32`, `ldlinux.e64`, `syslinux.cfg` (se você já tiver criado um).

6. **Criar ou editar o arquivo de configuração `syslinux.cfg`**
   O arquivo `syslinux.cfg` define o comportamento do menu de boot. Crie ou edite usando:
   ```bash
   sudo nano ~/easyimg_efi/syslinux.cfg
   ```

   Insira o seguinte conteúdo:
   ```cfg
   UI ldlinux.c32
   PROMPT 0
   MENU TITLE easyos Linux Boot Menu
   TIMEOUT 50
   DEFAULT linux

   LABEL linux
    MENU LABEL easyos Linux Live
    KERNEL /easyos/vmlinuz
    APPEND initrd=/easyos/initrd root=LABEL=easy2 rd.live.image

   LABEL linux-vesa
    MENU LABEL easyos Linux Live (VESA)
    KERNEL /easyos/vmlinuz
    APPEND initrd=/easyos/initrd root=LABEL=easy2 rd.live.image nomodeset xforcevesa
   ```

   **Nota:** `LABEL=easy2` corresponde ao nome da partição raiz, identificado anteriormente no comando `lsblk -f`.

7. **Grave o registro de boot mestre (MBR)**
   Certifique-se de que o **Master Boot Record (MBR)** está gravado na imagem. Use:
   ```bash
   sudo dd if=/usr/lib/syslinux/mbr.bin of=/dev/loop0 bs=440 count=1 conv=notrunc
   ```

8. **Desmonte e libere o dispositivo loop**
   Finalize o trabalho desmontando a partição e liberando o dispositivo loop:
   ```bash
   sudo umount ~/easyimg_efi
   sudo losetup -d /dev/loop0
   ```

9. **Grave a imagem no pendrive**
   Grave a imagem customizada no pendrive:
   ```bash
   sudo dd if=easy-6.6.10-amd64-custom.img of=/dev/sdb bs=4M status=progress oflag=sync
   sync
   ```
   **⚠️ Substitua `/dev/sdb` pelo seu dispositivo correspondente (pendrive).**

10. **Teste o boot**
    - Insira o pendrive no computador antigo.
    - Configure o BIOS para iniciar em **Legacy Mode** (não UEFI).
    - Certifique-se de priorizar o pendrive como dispositivo de boot.

---

## **Resolução de Problemas Comuns**
### **Erro: "Kernel not found"**
- Verifique se os arquivos `/boot/vmlinuz` e `/boot/initrd.img` existem na partição raiz (ext4, **LABEL=easy2**):
  ```bash
  sudo mount /dev/sdb2 /mnt
  ls /mnt/boot
  ```
  Caso não estejam lá, corrija os caminhos no arquivo `syslinux.cfg`.

### **Erro: Partição raiz não é montada**
- Substitua `LABEL=easy2` por `UUID=<UUID da partição>` no arquivo `syslinux.cfg`. O comando `lsblk -f` já identificou o UUID da partição (`e5fb950a-a51e-11f0-b7e6-287fcfeb4376`).

Atualize, por exemplo:
```cfg
APPEND initrd=/boot/initrd.img root=UUID=e5fb950a-a51e-11f0-b7e6-287fcfeb4376 rd.live.image
```

---

Com essas etapas concluídas, o **SYSLINUX** será configurado corretamente, e o sistema deve inicializar a partir do **pendrive** no PC antigo.

---

Com esses ajustes, o sistema deve inicializar corretamente. Caso haja dúvidas ou um comportamento inesperado, estarei aqui para ajudar! 😊
