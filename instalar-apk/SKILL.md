---
name: instalar-apk
description: Encontra o APK app-release.apk mais recente na pasta Downloads e instala no dispositivo Android conectado via USB. Ao final informa se a instalação foi bem-sucedida e exibe a data/hora do arquivo.
argument-hint: ""
---

# Skill: Instalar APK via USB

## O que essa skill faz
1. Localiza o arquivo `app-release.apk` mais recente na pasta Downloads do usuário
2. Verifica se há um dispositivo Android conectado via USB (ADB)
3. Instala o APK no dispositivo
4. Informa o resultado da instalação e a data/hora de modificação do arquivo

---

## Passo 1 — Verificar se ADB está disponível

Execute:
```
adb version
```

Se o comando falhar, informe o usuário que o ADB não está instalado e oriente a instalar o Android Platform Tools.

---

## Passo 2 — Verificar dispositivo conectado

Execute:
```
adb devices
```

- Se a lista estiver vazia ou só mostrar o cabeçalho `List of devices attached`, informe que nenhum dispositivo foi detectado e oriente o usuário a:
  - Conectar o celular via USB
  - Habilitar **Depuração USB** nas opções do desenvolvedor
  - Aceitar a permissão de depuração no celular

- Se houver dispositivo com status `unauthorized`, instrua o usuário a aceitar a permissão de depuração no celular.

- Se houver mais de um dispositivo, use o primeiro da lista.

---

## Passo 3 — Localizar o APK mais recente

> **Nota:** Esta etapa é apenas leitura — execute os comandos diretamente sem pedir confirmação ao usuário.

Execute o comando abaixo para listar todos os arquivos `app-release.apk` na pasta Downloads (incluindo subpastas) ordenados por data de modificação, do mais novo para o mais antigo:

```bash
ls -t ~/Downloads/**/app-release.apk ~/Downloads/app-release.apk 2>/dev/null | head -1
```

Se não encontrar, tente também com `find`:
```bash
find ~/Downloads -name "app-release.apk" -printf "%T@ %p\n" 2>/dev/null | sort -rn | head -1 | awk '{print $2}'
```

Se nenhum arquivo for encontrado, informe o usuário e peça que confirme o nome e local do arquivo.

Após encontrar o arquivo, capture também sua data/hora de modificação:
```bash
stat -c "%y" "<caminho_do_apk>"
```

No Windows (via bash/Git Bash), use:
```bash
date -r "<caminho_do_apk>"
```

---

## Passo 4 — Verificar se o app já está instalado e desinstalar

O package name do app é `com.onhappy.app`. Verifique se está instalado:

```bash
adb shell pm list packages | grep com.onhappy.app
```

- Se retornar resultado, o app está instalado — desinstale-o:

```bash
adb uninstall com.onhappy.app
```

- Se não retornar nada, pule este passo e vá direto para a instalação.

---

## Passo 5 — Instalar o APK

Execute:
```bash
adb install "<caminho_do_apk>"
```

- Se o dispositivo exibir um prompt de confirmação, oriente o usuário a aceitar no celular.
- Se retornar outros erros, exiba o erro completo e oriente conforme o código.

---

## Passo 6 — Reportar resultado

Ao final, exiba uma mensagem clara com:

- **Status:** Instalado com sucesso ✓ ou falha (com motivo)
- **Arquivo:** caminho completo do APK instalado
- **Data/hora do APK:** data e hora de modificação do arquivo
- **Dispositivo:** ID do dispositivo onde foi instalado

Exemplo de saída esperada:
```
APK instalado com sucesso.

Arquivo : /c/Users/lucas.santana_onfly/Downloads/app-release.apk
Data/hora: 2025-04-15 14:32:10
Dispositivo: R9XY502LAGA
```
