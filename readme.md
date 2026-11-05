# Seminário de Processamento Digital de Sinais e Imagens: Remoção de ruído em imagens — métodos clássicos vs. redes neurais

**Equipe:**
- 202504015 - Júlia Pereira Souza
- 202504003 - Gustavo Duarte
- 202503992 - Ana Lowe
- 202505970 - Jamily Vieira Gonçalves

**Apresentação:**
https://canva.link/0fazf3q2o2w89r7

## Resumo
 
Toda imagem capturada por um sensor carrega ruído, e removê-lo é uma das tarefas mais antigas do processamento de imagens. Hoje convivem dois caminhos: métodos clássicos, que são rápidos, não precisam de treino e são interpretáveis, e redes neurais, que costumam dar melhores resultados, mas exigem dados, computação e podem falhar quando o ruído do teste é diferente do ruído do treino. Este trabalho compara os dois caminhos em três cenários de ruído (gaussiano sintético, periódico sintético e real de câmera de celular), usando apenas bases públicas (BSD68 e SIDD) e redes com pesos pré-treinados (DnCNN e NAFNet). A pergunta é: **em que tipo de ruído um método clássico basta, e quando uma rede neural compensa o custo?**
 
## 1. Motivação
 
- **O problema é real e atual.** Câmeras de celular têm sensores pequenos, e em pouca luz o ruído fica visível. O SIDD foi criado justamente para estudar esse ruído.
- **A escolha entre clássico e rede tem custo.** Uma rede exige dados de treino, GPU e mais tempo por imagem. Se um filtro clássico resolve um tipo de ruído, esse custo não se justifica.
- **O tema une a disciplina ao nosso curso.** Os métodos clássicos usam convolução 2D (filtros espaciais) e DFT 2D (filtros na frequência), conteúdos de INF0413. As redes trazem a parte de IA.
- **Continuidade com a proposta anterior.** O ruído periódico (cenário C2) é o equivalente em imagem do zumbido de 60 Hz: fica concentrado em poucos pontos do espectro e pode ser removido com um filtro notch.
## 2. Contexto e revisão bibliográfica
 
| Ano | Referência | Contribuição para o trabalho |
| --- | --- | --- |
| 2001 | Martin et al. — Berkeley Segmentation Dataset | Origem das imagens do BSD68, usado em C1 e C2 |
| 2004 | Wang et al. — SSIM | Métrica de similaridade estrutural usada na avaliação |
| 2005 | Buades, Coll & Morel — Non-Local Means | Método clássico: média de trechos parecidos espalhados pela imagem |
| 2007 | Dabov et al. — BM3D | Método clássico de referência: agrupa blocos semelhantes e filtra no domínio da transformada |
| 2017 | Zhang et al. — DnCNN | Rede convolucional que aprende o ruído (aprendizado residual) e supera o BM3D em ruído gaussiano |
| 2018 | Abdelhamed, Lin & Brown — SIDD | Base de imagens com ruído real de cinco câmeras de celular, com gabarito |
| 2022 | Chen et al. — NAFNet | Rede simples, sem funções de ativação não lineares, com 40,30 dB de PSNR no SIDD |
| — | Gonzalez & Woods — Digital Image Processing | Filtros espaciais e filtros no domínio da frequência, incluindo o notch |
 
## 3. Pergunta e hipóteses
 
**Pergunta:** em que tipo de ruído um método clássico basta, e quando uma rede neural compensa o custo em dados, computação e complexidade?
 
- **H1 (gaussiano):** a vantagem da rede sobre o melhor clássico cresce com a intensidade do ruído.
- **H2 (periódico):** o filtro notch na DFT 2D supera os filtros espaciais e também uma rede treinada só em ruído gaussiano, porque o ruído do teste é diferente do ruído do treino.
- **H3 (real):** em ruído real de câmera, que não é gaussiano e depende da luz, a rede treinada em dados reais supera os clássicos, e a rede treinada em ruído sintético perde desempenho.
## 4. Metodologia
 
### 4.1 Visão geral (diagrama de blocos)
 
```
 1. Dados                     2. Ruído                           4. Métodos (mesmas imagens para todos)
 BSD68 (cinza) ─────────────► C1 gaussiano σ = 15, 25, 50  ───►  ┌─────────────────────────────────────┐
 SIDD validação (cor) ──────► C2 periódico, 3 amplitudes   ───►  │ Espaciais: média, gaussiano, mediana │
                              C3 real (já vem ruidoso)     ───►  │ Frequência: DFT 2D passa-baixa/notch │
                              semente fixa, salvo em disco       │ Estatísticos: Wiener, NLM, BM3D      │
                                                                 │ Redes pré-treinadas: DnCNN, NAFNet   │
 3. Ajuste de parâmetros                                         └──────────────────┬──────────────────┘
 Set12 (fora do teste) ────────────────────────────────────────────────────────────►│
                                                                                    ▼
                                         5. Avaliação: PSNR, SSIM, Wilcoxon, custo, análise visual
                                                                                    │
                                                                                    ▼
                                         6. Teste das hipóteses H1, H2 e H3 por cenário
```
 
*O diagrama desenhado está nos slides da apresentação.*
 
### 4.2 Dados e cenários
 
Todos os dados vêm de bases públicas. O ruído sintético segue o protocolo padrão da literatura de remoção de ruído.
 
| Cenário | Dados | Ruído | Por que este cenário |
| --- | --- | --- | --- |
| C1 | BSD68: 68 imagens em tons de cinza | Gaussiano branco, σ ∈ {15, 25, 50} (escala 0–255) | Protocolo padrão; o DnCNN foi avaliado exatamente assim |
| C2 | BSD68 (cinza) | Periódico: soma de duas senoides 2D (listras), com 3 níveis de amplitude | Ruído concentrado em poucos pontos da DFT 2D, onde o notch deve funcionar |
| C3 | SIDD, conjunto de validação: 1.280 blocos de 256×256, em cor (sRGB) | Real, de câmeras de celular | Ruído real com gabarito disponível, fora do treino do NAFNet |
 
- **Por que a validação do SIDD, e não o SIDD-Small.** O SIDD-Medium (320 pares) é usado para treinar redes como o NAFNet, e o conjunto de benchmark tem o gabarito oculto. A validação tem 1.280 blocos de 256×256 com gabarito e não entra no treino. Usar imagens do SIDD-Medium testaria o NAFNet em dados que ele já viu.
- **Reprodutibilidade.** O ruído de C1 e C2 é gerado uma única vez, com semente fixa, e salvo em disco. As frequências e orientações das senoides de C2 também são sorteadas uma vez com semente fixa. Todos os métodos recebem exatamente as mesmas imagens ruidosas.
- **Ajuste de hiperparâmetros.** Os parâmetros dos clássicos (tamanho do kernel, h do NLM, raio do notch etc.) são ajustados no Set12, que não faz parte do teste. Sem isso, a comparação favoreceria o método que teve mais ajuste.
### 4.3 Métodos
 
**Clássicos**
 
| Família | Métodos | Implementação |
| --- | --- | --- |
| Espaciais (convolução 2D) | Média, gaussiano, mediana | `scipy.ndimage` |
| Frequência (DFT 2D) | Passa-baixa gaussiano; notch automático | `np.fft.fft2` |
| Estatísticos | Wiener adaptativo, Non-Local Means, BM3D | `scipy.signal.wiener`, `skimage.restoration.denoise_nl_means`, pacote `bm3d` |
 
- **Notch automático.** Calcular o espectro de magnitude em escala logarítmica, ignorar um disco ao redor da origem (onde está a maior parte da energia da imagem), detectar os picos que ficam muito acima da mediana da vizinhança, zerar um disco ao redor de cada pico e do seu simétrico (−u, −v), e aplicar a DFT inversa. Os raios e o limiar de detecção são ajustados no Set12.
- **Wiener.** Usaremos o Wiener adaptativo local do `scipy.signal`. O `skimage.restoration.wiener` é uma deconvolução e não serve para este caso.
- **Nível de ruído.** NLM e BM3D precisam de σ. Em todos os cenários, σ é estimado da própria imagem (`skimage.restoration.estimate_sigma`). No C1, também reportamos o resultado com o σ verdadeiro, como referência.
**Redes (pesos pré-treinados)**
 
| Rede | Pesos | Treinada em |
| --- | --- | --- |
| DnCNN | `dncnn_15`, `dncnn_25`, `dncnn_50`, `dncnn_gray_blind`, `dncnn_color_blind` (repositório KAIR) | Ruído gaussiano sintético |
| NAFNet | NAFNet-SIDD width32 e width64 (repositório oficial) | Ruído real do SIDD-Medium |
 
- **Cinza e cor.** C1 e C2 são em cinza: o DnCNN usa os modelos de cinza, e no NAFNet a imagem cinza é replicada nos três canais e a saída é a média dos canais. C3 é em cor: o DnCNN usa o `dncnn_color_blind`, o BM3D usa a versão para RGB do pacote e os demais clássicos são aplicados por canal.
- **Cruzamento.** Todas as redes rodam nos três cenários. O DnCNN, treinado em ruído gaussiano, é avaliado no C2 e no C3; o NAFNet, treinado em ruído real, é avaliado no C1 e no C2. Esse cruzamento é o que testa H2 e H3.
- **Bônus (se houver tempo).** Treinar uma rede pequena no ruído periódico do C2, usando as 400 imagens de treino do DnCNN, para ver se a vantagem do notch desaparece.
### 4.4 Como cada hipótese será testada
 
| Hipótese | Cenário | Comparação | O que confirma a hipótese |
| --- | --- | --- | --- |
| H1 | C1, σ = 15, 25, 50 | Melhor rede vs. melhor clássico, em cada σ | A diferença de PSNR aumenta com σ |
| H2 | C2 | Notch DFT 2D vs. espaciais vs. DnCNN | O notch tem PSNR maior que os demais |
| H3 | C3 | NAFNet vs. clássicos vs. DnCNN color blind | O NAFNet vence, e o DnCNN cai em relação ao C1 |
 
### 4.5 Métricas e análise
 
- **PSNR e SSIM** por imagem (`skimage.metrics`), com média e desvio padrão por método e cenário.
- **Teste estatístico:** teste pareado de Wilcoxon (`scipy.stats.wilcoxon`) entre o melhor clássico e cada rede, por cenário, quando a diferença de médias for pequena. Nível de significância: 0,05.
- **Custo:** tempo por imagem (mediana de 3 execuções), em CPU e em GPU (Colab), e número de parâmetros das redes.
- **Análise qualitativa:** recortes com zoom em bordas e texturas, mapa de erro |x̂ − x| e espectro 2D (log da magnitude) antes e depois da filtragem.
## 5. Cronograma
 
Supomos 5 semanas a partir da prévia, com aulas às sextas. As datas devem ser ajustadas à data da entrega final.
 
| Semana | Período | Atividades | Entregável |
| --- | --- | --- | --- |
| 1 | 06/11 – 12/11 | Prévia (06/11). Ajustar a proposta. Montar o ambiente (scikit-image, scipy, `bm3d`, PyTorch, KAIR, NAFNet/BasicSR). Baixar BSD68, Set12 e a validação do SIDD. Gerar C1 e C2 com semente fixa. | Ambiente funcionando e dados prontos |
| 2 | 13/11 – 19/11 | Checkpoint 1 (13/11). Implementar os clássicos: espaciais, DFT 2D (passa-baixa e notch automático), Wiener, NLM e BM3D. Ajustar parâmetros no Set12. Montar o pipeline de avaliação. | Tabela dos clássicos em C1 e C2 |
| 3 | 20/11 – 26/11 | Rodar DnCNN e NAFNet nos três cenários. Rodar os clássicos no C3. Medir tempo e parâmetros. | Tabela completa: métodos × cenários |
| 4 | 27/11 – 03/12 | Testes de Wilcoxon, gráfico PSNR × σ (H1), figuras qualitativas e espectros. Responder H1–H3. Bônus, se houver folga. | Resultados e conclusões por hipótese |
| 5 | 04/12 – 10/12 | Escrever o relatório e os slides finais. Ensaiar a apresentação. | Entrega final |
 
## 6. Riscos e plano de contingência
 
| Risco | Prevenção | Se acontecer |
| --- | --- | --- |
| Download ou formato da validação do SIDD | Baixar e abrir os arquivos na semana 1 | Usar um subconjunto dos blocos, e registrar no relatório |
| Dependências do NAFNet (BasicSR) | Instalar na semana 1 | Rodar no Colab; em último caso, o C3 fica com os clássicos e o DnCNN |
| Falta de GPU | Blocos de 256×256 rodam em CPU | Usar o Colab para as redes |
| Atraso no cronograma | Bônus fica por último | O bônus é o primeiro corte |
 
## 7. Referências
 
1. ABDELHAMED, A.; LIN, S.; BROWN, M. S. A High-Quality Denoising Dataset for Smartphone Cameras. In: *IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)*, 2018.
2. BUADES, A.; COLL, B.; MOREL, J.-M. A non-local algorithm for image denoising. In: *IEEE Conference on Computer Vision and Pattern Recognition (CVPR)*, 2005.
3. CHEN, L.; CHU, X.; ZHANG, X.; SUN, J. Simple Baselines for Image Restoration. In: *European Conference on Computer Vision (ECCV)*, 2022.
4. DABOV, K.; FOI, A.; KATKOVNIK, V.; EGIAZARIAN, K. Image Denoising by Sparse 3-D Transform-Domain Collaborative Filtering. *IEEE Transactions on Image Processing*, v. 16, n. 8, 2007.
5. GONZALEZ, R. C.; WOODS, R. E. *Digital Image Processing*. 4. ed. Pearson, 2018.
6. MARTIN, D.; FOWLKES, C.; TAL, D.; MALIK, J. A database of human segmented natural images and its application to evaluating segmentation algorithms and measuring ecological statistics. In: *IEEE International Conference on Computer Vision (ICCV)*, 2001.
7. WANG, Z.; BOVIK, A. C.; SHEIKH, H. R.; SIMONCELLI, E. P. Image quality assessment: from error visibility to structural similarity. *IEEE Transactions on Image Processing*, v. 13, n. 4, 2004.
8. ZHANG, K.; ZUO, W.; CHEN, Y.; MENG, D.; ZHANG, L. Beyond a Gaussian Denoiser: Residual Learning of Deep CNN for Image Denoising. *IEEE Transactions on Image Processing*, v. 26, n. 7, 2017.
 
