# 📚 Arquitetura de Dublagem e Tradução: Deltarune

Este documento detalha o funcionamento dos dois principais scripts em Python criados para injetar sistemas de tradução externa e dublagem automática no código-fonte do Deltarune (via UndertaleModTool).

---

## 1. `auto_dubbing_patcher.py` (Capítulos Modernos: 2, 3, 4...)

Desenvolvido para lidar com a arquitetura mais moderna do GameMaker (versão 2.3+), onde os textos em inglês são embutidos nativamente no código através de funções como `msgsetloc` e `stringsetloc`. 

Este script não só injeta a dublagem, como **desacopla a localização do jogo**, permitindo que ele leia textos de fora.

### A. O Sistema `getstringlang` e o Arquivo `lang_en.json`
Como o jogo base não utiliza arquivos externos, o script varre o código procurando textos estáticos, guarda-os em um dicionário Python e substitui a string no código por uma chamada de função: `getstringlang("ID_DA_FALA")`. 
Em paralelo, ele gera e salva um arquivo `lang_en.json` físico contendo todos os textos.

Para que o jogo entenda esse comando, o script injeta a função `getstringlang` na inicialização do jogo:
1. **Verificação de Memória:** Checa se `global.lang_data` (o dicionário do jogo) já existe.
2. **Leitura de Disco:** Se não existir, ele procura pelo `lang/lang_en.json` físico na pasta do jogo, abre com `file_text_open_read` e armazena o conteúdo.
3. **Parseamento JSON:** Utiliza `json_parse()` nativo do GameMaker para converter as strings lidas em um objeto em memória.
4. **Retorno Seguro:** Quando o jogo tenta escrever uma fala na tela, a função busca a chave dentro desse objeto JSON e retorna o texto traduzido. Se falhar, ela simplesmente retorna o ID como *fallback*.

### B. O Reprodutor de Áudio (`scr_play_dubbing_audio`)
O script moderno injeta o motor de áudio como uma `function` global isolada, injetada de forma segura na *engine*.

1. **Captura do ID:** Ele altera silenciosamente a função `msgsetloc` (que exibe os textos) para guardar o ID do texto atual na variável `global.msg_id[msg_index]`.
2. **Corte Rápido (Skipping):** Se um áudio antigo ainda estiver tocando (`global.current_dub_stream`), ele o destrói imediatamente. Isso garante que a fala anterior seja interrompida se o jogador apertar 'Z' para pular o texto rápido.
3. **Regex Interno (Tratamento do ID):** Se o ID capturado for `obj_joker_slash_Step_0_gml_35_0`, a engine procura o primeiro `_`, corta o começo e adiciona `snd_`. O jogo vai procurar o arquivo `dubs/snd_joker_slash_Step_0_gml_35_0.ogg`.
4. **Reprodução e Ganho:** Verifica se o arquivo físico existe e o reproduz como um *stream* assíncrono. Em seguida, aplica um `audio_sound_gain` de `3.0` (300% de volume) para garantir que as vozes não sejam engolidas pelos efeitos sonoros do jogo.

### C. O Sistema de "Ducking" (Abaixar a Música)
Para garantir um tom cinematográfico, o script busca o arquivo de processamento de tempo do jogo (`obj_time_Step`), que roda todos os frames, e injeta o monitoramento do áudio:
- Ele checa se `audio_is_playing(global.current_dub_stream)` é verdadeiro.
- **Se a voz está tocando:** Aplica um `audio_sound_gain` de `0.4` (40%) no `global.currentsong[1]` (Música ambiente) e no `global.batmusic[1]` (Música de batalha), num fade de 300ms.
- **Se a voz parar (ou o balão fechar):** Retorna ambas as trilhas para `1.0` (100%) em um fade de volta suave.

---

## 2. `patch_apenas_dublagem_ch1.py` (Capítulo 1 Clássico)

Diferente do Capítulo 2, o Capítulo 1 foi construído em uma engine GMS mais antiga e já foi preparado para localização oficial desde o lançamento (portanto, ele já usa nativamente o `scr_84_get_lang_string` e já tem o seu próprio `lang_en.json`). O objetivo deste script é **exclusivo para injeção cirúrgica de dublagem** driblando limitações de compilação.

### A. Driblando a Compilação do UndertaleModTool
Muitos dos diálogos no Capítulo 1 são ativados em condições simples de apenas uma linha no código:
```gml
if (condicao)
    global.msg[0] = scr_84_get_lang_string("ID_AQUI");
```
Tentar injetar uma segunda instrução (`global.msg_id[0] = ...`) nessa linha causava um erro fatal de sintaxe na ferramenta. O script em Python resolve isso de forma elegante, convertendo a linha perfeitamente para um bloco seguro de compilação usando chaves `{ }`:
```gml
if (condicao)
    { global.msg[0] = scr_84_get_lang_string("ID_AQUI"); global.msg_id[0] = "ID_AQUI"; }
```

### B. Injeção de Código em Texto Bruto (Inline)
No GMS antigo, não é possível anexar facilmente funções globais limpas (`function nome()`) em arquivos alheios. Para garantir 100% de estabilidade, este script apaga a chamada do reprodutor do Undertale e **cola o bloco inteiro do código de reprodução (incluindo tratamento do arquivo, checagem e streams) diretamente no corpo do `scr_textsound.gml`**. 

### C. Correção do Pulo de Falas (`last_dub_played`)
Como o Capítulo 1 destrói o objeto de texto constantemente, o sistema usa `id.last_dub_played` em vez de uma variável global. Isso resolve um bug grave da engine clássica onde, se uma caixa de texto da cena anterior encerrasse na frase "0", a primeira frase ("0") do próximo personagem seria pulada em absoluto silêncio pelo sistema. Ao vincular o histórico à instância (`id`), a memória da voz reseta toda vez que um balão de fala novo se abre.
