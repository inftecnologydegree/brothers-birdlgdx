# 📑 Relatório de Análise de Causa Raiz (RCA) - Gerenciamento de Áudio no LibGDX (Web/GWT)

Este documento registra a investigação e a resolução de um bug crítico de comportamento assíncrono de áudio identificado na versão Web (HTML5/GWT) do projeto, cuja falha não era replicada no ambiente nativo Android.

---

## 🛑 Descrição do Problema
O efeito sonoro contínuo (`fireBurning`) associado a um objeto dinâmico na tela rolante entrava em loop infinito no navegador web. O áudio falhava em responder aos comandos de interrupção disparados pelos gatilhos de ciclo de vida do objeto (descarte ao sair da tela) ou por eventos de entrada do usuário (clique de botões), continuando a tocar indefinidamente.

---

## 🔍 Investigação e Análise de Hipóteses

### 🏆 Hipótese 1: Rigidez de Ciclo de IDs e Conversão Cross-Compiler GWT (Causa Raiz)
* **Teoria:** O backend HTML5 do LibGDX utiliza o Google Web Toolkit (GWT) para traduzir código Java para JavaScript. A API de áudio web gerada é estritamente dependente da captura e manipulação cronológica exata do identificador do canal de som (`soundId`).
* **Mapeamento da Falha:** No código original, o método `setLooping(soundId, true)` era chamado *antes* que a função `.play()` atribuísse um novo ID válido à variável. Adicionalmente, chamadas globais de interrupção como `.stop()` (sem especificação de canal) causavam desiteração no Web Audio API do navegador, fazendo com que a instância de áudio em loop perdesse sua referência e continuasse alocada em memória.

### 📉 Hipótese 2: Variáveis Nuláveis Não Tratadas (Menos Provável)
* **Teoria:** Falha na verificação de nulidade (`NullPointerException`) ou ciclo de vida de objetos nulos durante a reciclagem e descarte das entidades na tela rolante, impedindo que o fluxo de execução alcançasse o método `.stop()`.
* **Validação:** Descartada após análise de logs. O fluxo de renderização e descarte mantinha consistência de referências. O método de pausa era invocado com sucesso, porém o motor de áudio web ignorava o comando, confirmando que a falha residia no subsistema de som e não na estrutura de estados do jogo.

---

## 🛠️ Solução Aplicada

A correção exigiu a reestruturação da ordem de execução do gerenciamento de canais de áudio, garantindo a captura síncrona do `soundId` antes de qualquer manipulação de propriedades, além do escopo explícito de encerramento do canal.

### Código Corrigido no Game Manager:

```
java
private long soundId = -1; 

public void playBurn(float volume) {
    if (fireBurning != null) {
        // 1. Stops residual ghost instances before overlaying the channel.
        if (soundId != -1) {
            fireBurning.stop(soundId);
        }
        
        // 2. Perform playback first to register a valid soundId on the Web.
        soundId = fireBurning.play(volume, 4.1f, 0);
        
        // 3. Applies the loop exclusively to the verified active ID.
        fireBurning.setLooping(soundId, true);
        
        soundCooldown = 0.015f; 
    }
}

public void pauseFire() {
    if (fireBurning != null && soundId != -1) {
        // Disables the loop persistence flag.
        fireBurning.setLooping(soundId, false);
        
        // It concludes execution by surgically targeting the mapped ID.
        fireBurning.stop(soundId); 
        
        soundId = -1; 
    }
}
```

---

## 📈 Lições Aprendidas e Conclusão
1. **Divergência de Backends:** Plataformas nativas (Android/JVM) gerenciam referências de áudio com tolerância a encavalamento de chamadas estáticas, enquanto engines baseadas em browsers exigem controle estrito e sequencial de ponteiros e IDs.
2. **Abstração Cross-Compiler:** Ao programar com LibGDX para Web, propriedades de mutação de estado de um recurso (como `.setLooping()` ou `.setVolume()`) só devem ocorrer em momentos subsequentes à instanciação real do stream gerada pelo método `.play()`.

A aplicação desta abordagem restabeleceu o comportamento multiplataforma idêntico do jogo, eliminando vazamentos de memória de áudio e garantindo a portabilidade total estável da build web hospedada via Vercel.
