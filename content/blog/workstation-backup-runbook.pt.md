---
title: "Configurando backups (estação de trabalho)"
date: 2022-12-18T18:56:16-03:00
draft: false
situação: "Validado"
duration: "1 hora"

---

# Propósito

Configurar o programa de backup para replicar dados para nuvem e aplicar proteção contra ransomware.

Este procedimento satisfaz a [regra de ouro 3-2-1-1-0](https://web.archive.org/web/20221108104150/https://community.veeam.com/blogs-and-podcasts-57/3-2-1-1-0-golden-backup-rule-569) e protege os dados do usuário contra **ameaças**:

1. Falha humana (esquecimento, descuido, desconhecimento, etc)
2. Roubo e/ou furto
3. Falha de mídia (física e/ou lógica)
4. Catástrofe local (incêndio, inundação, etc)
5. Malware (ransomware)

Validar o procedimento pressupõe sucesso em:

- Recuperação pontual de determinado arquivo em versão específica, sem necessitar de acesso à internet.
- Recuperação de todos os dados do usuário para um novo dispositivo usando cópia em nuvem ou local, como mídia removível ou [NAS](https://web.archive.org/web/20221218233749/https://www.qnapbrasil.com.br/blog/post/o-que-e-nas-network-attached-storage).


# Contexto

Softwares e fornecedores utilizados:

- O [Borgbackup](https://www.borgbackup.org/) é usado para backup **incremental**;
- O [Vorta](https://vorta.borgbase.com/) é usado como agendador e fornece uma interface gráfica para interação com o Borg;
- O [Rclone](https://rclone.org/) é usado para sincronização entre os repositórios no sistema de arquivos local e o armazenamento em nuvem;
- A [Backblaze](https://www.backblaze.com/b2/cloud-storage.html) é usado como fornecedor de armazenamento em nuvem;
- O [ntfy.sh](https://ntfy.sh) é usado como canal de notificação com o resultado de cada execução de backup;

Depois de configurado, o Vorta executa na área de trabalho em segundo plano e, no horário agendado, inicia a rotina de backup. Em caso de sucesso, o script de pós-backup é executado também.

A rotina de backup é operada pelo Borg, que salva uma versão de todos os arquivos selecionados dentro do repositório local (no mesmo sistema de arquivos)

O script de pós-backup sincroniza o repositório local com a nuvem, aumenta a validade da proteção contra ransomware e por fim envia uma notificação com o resultado da execução.


# Passo-a-passo

Em um sistema Debian stable, instale os programas abaixo:

`apt install vorta borg rclone backblaze-b2 curl python3`

Apenas para referência, abaixo está a versão específica dos softwares:

- backblaze-b2 1.3.8-4
- borg 1.1.16-3
- curl 7.74.0-1.3+deb11u3
- debian stable 11.6
- linux-image-5.10.0-20-amd64 (5.10.158-2)
- python3 3.9.2-3
- rclone 1.53.3-1+b6
- vorta client 0.7.5-1

Nota: O procedimento foi validado com o sistema operacional e softwares mencionados acima, mas esse procedimento provavelmente funciona em qualquer sistema linux ou demais sistemas suportados pelo Borg Backup.


## Vorta Profile

Execute o programa Vorta e crie seu repositório local. Esta página tem um guia de 5 minutos para fazê-lo a partir do passo 2: [Setting up Vorta for Local Backups](https://web.archive.org/web/20221217124626/https://vorta.borgbase.com/usage/local/#step-2---setting-up-local-repository)

Após configurar o repositório, ajuste as demais configurações do seu perfil como a seguir:

**Schedule tab/Schedule**

> 🔘 Backup every **1** hours at **17** minutes past the hour
> 
> ✅ Validate repository data every **1** weeks
> 
> ✅ Prune old Archives after each backup

Nota: A configuração acima é exemplificativa. Caso queira, pode alterar os números em negrito.

**Schedule tab/Shell Commands**

> Pre-backup command to run BEFORE backups: `test -x /home/user/.local/share/Vorta/post-backup_init`
>
> Post-backup command to run AFTER backups: `/home/user/.local/share/Vorta/post-backup_init 2>&1 | tee -a /home/user/.cache/Vorta/log/vorta.log`

Nota: substitua a palavra `user` nos comandos acima pelo nome do seu usuário.

Dica: em um terminal, execute `echo $LOGNAME` para conferir o nome do seu usuário.

> Arguments to add: `--umask 0022 --noatime`

**Archives tab/Prune Options and Archive Naming**

> Prune Options: Keep **6** hourly, **3** daily, **4** weekly, **2** monthly and **0** annual archives. No matter what, keep all archives of the last **2H**.
>
> Archive Name: `{profile_slug}-{now:%Y-%m-%d-%H%M}`
>
> Prune Prefix: `{profile_slug}-`

Nota: A configuração acima é exemplificativa. Caso queira, pode alterar os números em negrito.

## Instalar scripts personalizados

Junto com este documento encontram-se os arquivos post-backup_init e post-backup_renew-lock.py. Copie-os para o caminho /home/user/.local/share/Vorta/

Após a cópia, aplique permissões de execução no arquivo post-backup_init, como abaixo:

`chmod +x /home/$LOGNAME/.local/share/Vorta/post-backup_init`

Nota: A pasta .local é oculta por padrão. Caso necessário, habilite a visualização de arquivos ocultos no seu gerenciador de arquivos.

## Obter token válido na backblaze

Primeiro, crie um bucket exclusivo para os backups deste dispositivo. Escolha um nome globalmente único, de preferência seguindo alguma padronização interna (tal como nomedaempresa-hostname-backups). 

Lembre-se de habilitar a opção de trava de objetos! Como não pretendo compartilhar esses arquivos com ninguém fora da minha organização e eu sempre aplico criptografia no lado do cliente, mantive essas opções no padrão. As opções de criação do bucket ficaram assim:

> Files in Bucket are: Private
>
> Default Encryption: Disable
>
> Object Lock: Enable

Após criado, ajuste o ciclo de vida dos objetos para remover automaticamente versões antigas.

> Lifecycle Settings: Keep only the last version of the file

Segundo, criar uma nova application key com acesso restrito ao bucket criado acima. Habilitar leitura e escrita.

As credenciais do token serão exibidas na tela. Salve-as em algum lugar seguro pra uso futuro, ou aproveite agora para configurar o rclone. Recomendo usar a [documentação oficial](https://rclone.org/commands/rclone_config_create/).

## Receber notificações no celular (opcional)

Essa é a parte mais legal na minha opinião. Se você fez tudo certinho até aqui, o seu sistema **já está gerando notificações** ao término de cada backup, seja ele bem sucedido ou não. Para acompanhar essas notificações no seu celular, usei um serviço chamado ntfy.sh.

Baixe o aplicativo pelo F-droid (ou pela fonte de sua preferência) e cadastre o acompanhamento do tópico com o nome igualzinho aparece no seu aquivo de configuração do rclone. Como aqui usei o padrão nomedaempresa-hostname-backup, fica fácil acertar o nome do tópico para acompanhar.

Você pode visualizar suas configurações do rclone com o comando:

`rclone config show`

O nome da configuração usada deve aparecer entre colchetes.
