# Seminário Processamento Digital de Sinais e Imagenss: Remoção do zumbido de 60 Hz com a DFT

**Equipe:**
- 202504015 - Júlia Pereira Souza
- 202504003 - Gustavo Duarte
- 202503992 - Ana Lowe
- 202505970 - Jamily Vieira Gonçalves

**Apresentação:**
- https://canva.link/i2ugoixv118d7cg

## Resumo

O zumbido de 60 Hz, causado pela rede elétrica no Brasil, aparece com frequência em gravações caseiras e em sinais biomédicos como o ECG. Este trabalho propõe removê-lo com a DFT: o sinal é transformado para o domínio da frequência, as componentes de 60 Hz e seus harmônicos são zeradas e o sinal é recuperado pela DFT inversa. Usaremos uma gravação de voz limpa com zumbido adicionado artificialmente, o que permite comparar o resultado com a referência. Além do caso ideal, em que o zumbido cai exatamente sobre as raias da DFT, testaremos um zumbido com frequência levemente deslocada, como acontece na rede real. Esperamos que a remoção seja quase completa no caso ideal e que, no caso deslocado, seja preciso zerar uma faixa maior de raias, o que remove também parte da voz. Medir esse compromisso é o foco da avaliação.

## Introdução

### Motivação

Sinais captados perto de equipamentos elétricos quase sempre carregam interferência da rede. No Brasil, essa interferência ocorre em 60 Hz, com harmônicos em 120 Hz, 180 Hz e assim por diante. Em gravações de voz, ela aparece como um zumbido constante. Em eletrocardiogramas, pode mascarar detalhes do sinal clínico. O problema é simples de descrever e bom para aplicar os conceitos da disciplina: amostragem, DFT, magnitude, fase e síntese pela DFT inversa.

### Revisão bibliográfica

As referências abaixo sustentam a discussão.

1. **Levkov et al. (2005)**: revisão do procedimento de subtração para remoção da interferência da rede elétrica no ECG.
2. **Oppenheim & Schafer (Discrete-Time Signal Processing)**: base teórica da DFT, da DFT inversa e do efeito de vazamento espectral (*leakage*) quando a frequência não cai exatamente sobre uma raia.
3. **Smith (1997), The Scientist and Engineer's Guide to DSP**: explica a DFT e a filtragem de sinais de forma acessível, com exemplos práticos.
4. **Prandoni & Vetterli (2008), Signal Processing for Communications**: apresenta sinais discretos, a DFT e sua interpretação em magnitude e fase, conceitos que fundamentam o método proposto.
5. **Lyons (Understanding Digital Signal Processing)**: aborda filtros rejeita-faixa (*notch*), uma alternativa para remover interferências senoidais.

### Objetivos

- Aplicar a DFT e a DFT inversa para remover uma interferência senoidal de frequência conhecida.
- Avaliar a remoção com métricas objetivas e teste de escuta.
- Investigar o vazamento espectral quando o zumbido não cai exatamente sobre uma raia.
- Medir o compromisso entre remover o zumbido e preservar a voz.

## Metodologia

### Conjunto de dados (dataset)

Usaremos uma gravação de voz limpa, em ambiente silencioso, feita com o celular ou obtida de um banco de áudio com licença adequada.

**Reamostragem.** Celulares costumam gravar em 44,1 kHz ou 48 kHz. Reduziremos a taxa para fs = 8 kHz, suficiente para voz. Antes da redução, aplicaremos um filtro passa-baixa anti-aliasing com corte abaixo de 4 kHz (metade da nova taxa), para que frequências altas não se dobrem sobre a faixa da voz.

Sobre essa gravação, somaremos artificialmente um zumbido de frequência fundamental f_z com dois harmônicos:

> x_ruidoso[n] = x_limpo[n] + A₁·sen(2π·f_z·n/fs) + A₂·sen(2π·2f_z·n/fs) + A₃·sen(2π·3f_z·n/fs)

Como a referência limpa existe, é possível medir o resultado. Como etapa opcional, testaremos uma gravação real com zumbido.

Antes do processamento, faremos uma **análise exploratória** com a forma de onda, o espectro e a estimativa de energia em 60 Hz, para confirmar a presença do zumbido e escolher os parâmetros.

**Comprimento do sinal (N).** Faremos uma única DFT sobre a gravação inteira. Para que 60 Hz caia exatamente sobre uma raia, N·60/fs precisa ser inteiro. Com fs = 8 kHz, isso vale para qualquer N múltiplo de 400 amostras, então basta cortar a gravação para um múltiplo de 400. Exemplo: 5 s de áudio dão N = 40 000, resolução Δf = fs/N = 0,2 Hz e raias k = 300 (60 Hz), k = 600 (120 Hz) e k = 900 (180 Hz). Isso evita o vazamento espectral, que espalharia a energia do zumbido por raias vizinhas.

### Experimentos

| Experimento | Zumbido | Objetivo |
| --- | --- | --- |
| E1 — Caso ideal | f_z = 60 Hz exatos | Verificar o método quando o zumbido cai sobre as raias |
| E2 — Frequência deslocada | f_z = 60,1 Hz | Simular a variação da rede, com o sistema ajustado para 60 Hz |
| E3 — Gravação real (opcional) | Zumbido real | Testar o método sem referência limpa |

No E2, com Δf = 0,2 Hz, o zumbido fica meia raia distante de k = 300, uma raia distante de k = 600 e uma raia e meia distante de k = 900. Por isso, compararemos três larguras de remoção: zerar 1, 3 ou 5 raias em torno de cada harmônico.

### Modelo (sistema de processamento)

1. Calcular a DFT do sinal ruidoso: X_r[k].
2. Calcular as raias nominais do zumbido: k_m = m·60·N/fs, para m = 1, 2, 3.
3. Zerar as raias de k_m − d até k_m + d **e as raias espelhadas (N − k)**, para que o sinal recuperado continue real. No E1, d = 0. No E2, d = 0, 1 e 2 (1, 3 ou 5 raias).
4. Aplicar a DFT inversa e obter y[n].

### Métricas

- **SNR** em relação à voz limpa (E1 e E2). Antes da filtragem, usa-se o sinal ruidoso x_r[n] no lugar de y[n]:

  > SNR = 10·log₁₀( Σ x²[n] / Σ (y[n] − x[n])² )

- **Perda da voz** (E1 e E2): fração da energia da voz limpa que estava nas raias zeradas, calculada sobre a DFT X[k] da voz limpa. Mede diretamente o custo da remoção.

  > P = Σ_{k zeradas} |X[k]|² / Σ_k |X[k]|²

- **Redução da energia nas faixas do zumbido** (E3): energia do sinal em ±2 Hz ao redor de 60, 120 e 180 Hz, antes e depois, em dB. É a métrica possível quando não há referência limpa.
- **Comparação dos espectros** do sinal original, ruidoso e filtrado.
- **Teste de escuta** com a turma, comparando as versões.

Não usamos a magnitude de uma única raia, |X[k₀]|, como métrica de atenuação: depois da filtragem ela vale exatamente zero, e a atenuação em dB seria infinita.

### Fluxograma

```
Gravação de voz (44,1 ou 48 kHz)
      │
      ▼
Passa-baixa anti-aliasing + reamostragem para 8 kHz → x[n]
      │
      ▼
Cortar para N múltiplo de 400 amostras
      │
      ▼
Somar zumbido f_z, 2f_z, 3f_z  (E1: 60 Hz · E2: 60,1 Hz)  ──►  x_r[n]
      │
      ▼
DFT → X_r[k]
      │
      ▼
Zerar raias k_m ± d e espelhos (N − k)
      │
      ▼
DFT inversa → y[n]
      │
      ▼
Métricas: SNR, perda da voz, espectros, escuta
```

*O diagrama de blocos desenhado está nos slides da apresentação.*

## Discussão

A DFT permite atacar o zumbido de forma direta, porque ele é uma senoide estável e de frequência conhecida, concentrada em poucas raias. Zerar essas raias remove a interferência sem alterar as demais frequências. Isso se conecta à literatura: Levkov et al. (2005) tratam da remoção da interferência da rede elétrica no ECG, e o tema mostra um caminho alternativo no domínio da frequência.

O ponto central é o **custo da remoção**. No caso ideal (E1), esse custo é pequeno: com Δf = 0,2 Hz, cada raia zerada corresponde a uma faixa estreitíssima, e 60 Hz fica abaixo da frequência fundamental típica da voz. O compromisso aparece quando o zumbido não cai sobre as raias nominais (E2 e gravações reais). Para capturar sua energia, é preciso zerar uma faixa mais larga, que remove também parte da voz, sobretudo em 120 e 180 Hz, onde vozes graves têm energia. A perda da voz e o teste de escuta mostram até onde vale alargar essa faixa.

Uma alternativa é o filtro rejeita-faixa (*notch*), discutido por Lyons. Zerar raias da DFT já atua numa faixa mais estreita que um notch típico, mas exige o sinal inteiro antes de processar. O notch processa amostra por amostra, o que permite uso em tempo real, e sua largura de banda pode ser ajustada para tolerar pequenas variações na frequência da rede.

Outro cuidado é a escolha de N. Se 60 Hz não cair sobre uma raia, o zumbido se espalha em raias vizinhas e a remoção fica incompleta. Isso ilustra o efeito de vazamento espectral descrito por Oppenheim & Schafer, e o E2 mostra esse efeito na prática.

## Conclusão

[Preencher após os experimentos: em que medida os objetivos foram realizados. Incluir SNR antes e depois (E1 e E2), perda da voz para cada largura de remoção, resultado do E3, se realizado, e avaliação do teste de escuta.]

Hipótese a confirmar: no caso ideal, a DFT remove o zumbido quase sem afetar a voz; quando a frequência da rede varia, é preciso escolher entre remover mais ruído e preservar a voz.

## Referências

1. OPPENHEIM, A. V.; SCHAFER, R. W. *Discrete-Time Signal Processing*. Pearson.
2. PRANDONI, P.; VETTERLI, M. *Signal Processing for Communications*. EPFL Press, 2008.
3. SMITH, S. W. *The Scientist and Engineer's Guide to Digital Signal Processing*. California Technical Publishing, 1997.
4. LYONS, R. G. *Understanding Digital Signal Processing*. Prentice Hall.
5. LEVKOV, C. et al. Removal of power-line interference from the ECG: a review of the subtraction procedure. *BioMedical Engineering OnLine*, v. 4, art. 50, 2005.
