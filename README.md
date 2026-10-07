# Laboratório — a mesma loja, duas arquiteturas

Estilos Arquiteturais V · Microsserviços 

Nomes: randsonbredley

Vocês vão rodar a **mesma loja** de dois jeitos: como um **monolito** (um processo, um banco) e como **microsserviços** (quatro processos, um banco por serviço). As duas versões têm as mesmas rotas. Ninguém precisa programar — só um experimento pede para mudar um número no código. 

Requisito: Python 3.8 ou superior. Nada para instalar. No macOS/Linux use `python3`.

| Versão | Como subir | Endereço |
|---|---|---|
| Monolito | `python monolito.py` | http://localhost:8000 |
| Microsserviços | `python iniciar_microsservicos.py` | http://localhost:9000 |

O objetivo é descobrir, na prática, **onde os microsserviços ganham e onde eles cobram** . 

|**Versão**|**Endereço**|**Onde ficam os dados**|
|---|---|---|
|Monolito|http://localhost:8000|dados/monolito.json|
|Microsserviços|http://localhost:9000 (Vitrine)|dados/estoque.json e<br>dados/pedidos.json|
|Rotas nas duas| `/produto/1`, `/comprar/1`, `/relatorio`, `/bug/estoque`||
|Derrubar um serviço|http://localhost:<porta>/desligar|Catálogo 9001 · Estoque 9002 ·<br>Pedidos 9003|

**Faça** Abra dois terminais na pasta laboratorio. 

<mark># terminal 1 — leva 10 s para subir</mark> `python monolito.py` 

<mark># terminal 2 — sobe 4 processos</mark> `python iniciar_microsservicos.py` 

**Faça** Abra duas abas no navegador, lado a lado: 

http://localhost:8000/produto/1,  http://localhost:9000/produto/1 

**Deve aparecer** O mesmo produto nas duas: Teclado mecânico, R$ 250, 5 em estoque. 

### **Experimento 1 · Deploy de uma promoção** 

O time de vendas quer 10% de desconto. Vocês vão "publicar" essa mudança nas duas versões e observar **o que sai do ar enquanto isso**. 

**Faça — monolito a)** Em monolito.py, mude DESCONTO = 0 para DESCONTO = 10. **b)** No terminal 1, Ctrl+C e rode `python monolito.py` de novo. **c)** Durante os 10s de subida, tente abrir <localhost:8000/relatorio>. 

**Faça — microsserviços a)** Em catalogo.py, mude DESCONTO = 0 para DESCONTO = 10. **b)** Abra <localhost:9001/desligar> para derrubar só o Catálogo. **c)** Num terceiro terminal, rode `python catalogo.py`. **d)** Durante os 4s de subida, tente <localhost:9000/relatorio> e <localhost:9000/produto/1>.   

**Deve aparecer** Depois da subida, o preço é R$ 225 nas duas. 

||**Monolito**|**Microsserviços**|
|---|---|---|
|Quanto tempo ficou fora do ar?|10 segundos|4 segundos (apenas para o Catálogo)|
|O que parou de funcionar?|A loja inteira (todas as rotas da porta 8000)|Apenas as rotas que dependem do Catálogo (`/produto/1`, `/comprar/1`)|
|O que continuou funcionando?|Nada (0 rotas)|A rota `/relatorio` continuou respondendo normalmente|

**Responda** Poder ou problema dos microsserviços? Por quê? 
É um **poder** (vantagem). Permite **deploy independente** e **isolamento de falhas**: a atualização do serviço de Catálogo causou indisponibilidade parcial e mais curta (4s vs 10s), afetando apenas as rotas associadas enquanto outros serviços e rotas (como `/relatorio`) permaneceram totalmente operacionais.

### **Experimento 2 · Latência** 

**Faça** Num terminal livre, rode: 

`python comparar.py` 

**Faça** Abra de novo /produto/1 nas duas abas e compare o campo tempo_interno_ms. 

||**Monolito**|**Microsserviços**|
|---|---|---|
|Tempo médio por página (comparar.py)|~5.77 ms|~15.68 ms|
|tempo_interno_ms|~0.5 ms|~45.8 ms|

**Responda** Aqui tudo roda no mesmo computador. O que aconteceria com essa diferença se cada serviço estivesse numa máquina diferente? 
A diferença aumentaria significativamente. Em máquinas separadas ou ambientes distribuídos na nuvem, cada chamada entre microsserviços realiza um salto de rede (HTTP/TCP) com latência física de RTT (Round Trip Time) de vários milissegundos. Enquanto o monolito faz chamadas diretas em memória (nanossegundos), a arquitetura de microsserviços acumula latências de rede a cada serviço intermediário consultado.

### **Experimento 3 · Consistência** 

Na demonstração, o professor derrubou o Estoque e a loja em microsserviços **continuou vendendo** . Agora vocês vão ver o preço disso. 

**Faça a)** Abra <localhost:9000/relatorio> e anote os números. **b)** Derrube só o Estoque: <localhost:9002/desligar>. **c)** Compre duas vezes: <localhost:9000/comprar/1>. **d)** Religue o Estoque: `python estoque.py`. **e)** Abra <localhost:9000/relatorio> de novo. 

**Deve aparecer** Em (c): "aviso": "Estoque fora do ar: pedido registrado SEM baixa de estoque". Em (e): "pedidos_pendentes": 2 e "consistente": false. 

|**/relatorio (microsserviços)**|**Antes**|**Depois**|
|---|---|---|
|pedidos_confirmados|0|0|
|pedidos_pendentes|0|2|
|baixas_de_estoque|0|0|
|consistente|true|false|


**Faça** Abra a pasta dados/. Compare pedidos.json e estoque.json: cada banco conta uma história diferente. 

**Responda** No monolito, o bug do Estoque derrubou a loja inteira: nenhuma venda, mas nenhum dado errado. Nos microsserviços, a loja vendeu, mas os bancos discordam. **Qual dos dois a loja prefere? Quem decide isso?** 
Depende da estratégia e do modelo de negócio (Trade-off do Teorema CAP entre Disponibilidade e Consistência):
- Lojas de e-commerce costumam preferir a abordagem dos **Microsserviços (Priorizar Disponibilidade / Consistência Eventual)**, pois preferem garantir o pedido no carrinho e efetuar a reconciliação do estoque de forma assíncrona do que perder a venda no momento do checkout.
- Bancos e sistemas financeiros preferem o **Monolito (Priorizar Consistência Forte / ACID)**, onde é inadmissível ter dados inconsistentes ou saldos incorretos, optando por recusar a operação caso algum serviço falhe.
- **Quem decide isso?** Os **stakeholders de negócio** (Product Owners, Gerentes de Produto, diretores de negócio) em conjunto com a equipe de arquitetura de software.

### **Experimento 4 · Operação** 

**Faça** Faça uma compra em cada versão (/comprar/2 nas duas) e olhe os terminais. 

|**Para uma compra…**|**Monolito**|**Microsserviços**|
|---|---|---|
|Quantos processos estão rodando?|1 processo|4 processos (Vitrine, Catálogo, Estoque, Pedidos)|
|Quantas portas?|1 porta (8000)|4 portas (9000, 9001, 9002, 9003)|
|Quantos arquivos de banco?|1 arquivo (`monolito.json`)|2 arquivos (`estoque.json` e `pedidos.json`)|
|Quantas linhas de log apareceram?|1 linha|4 linhas|
|Em quantos serviços?|1 serviço|3 serviços (Vitrine, Catálogo, Pedidos/Estoque)|


**Responda** Se a compra desse errado, onde vocês procurariam o erro em cada versão? 
- **Monolito:** Procuraria em um único console/arquivo de log do processo rodando na porta 8000, ou examinando a função `pedidos_comprar` que executa todo o fluxo de forma centralizada e síncrona.
- **Microsserviços:** Seria necessário verificar e correlacionar logs de múltiplos serviços independentes (Vitrine na 9000, Pedidos na 9003, Catálogo na 9001 e Estoque na 9002), o que exige ferramentas de monitoramento centralizado e rastreamento distribuído (*distributed tracing* com Correlation IDs).

## **Placar da dupla** 

|**Experimento**|**Quem saiu melhor?**|**Para microsserviços, é poder ou problema?**|
|---|---|---|
|Bug fatal (demonstração)|Microsserviços|Poder (Isolamento de falhas: a queda do Estoque não derruba o restante da loja)|
|1 · Deploy|Microsserviços|Poder (Deploy independente com menor tempo fora do ar: 4s vs 10s)|
|2 · Latência|Monolito|Problema (O overhead dos saltos de rede entre serviços aumenta o tempo de resposta)|
|3 · Consistência|Monolito|Problema (A consistência eventual introduz o risco de dados divergentes entre bancos)|
|4 · Operação|Monolito|Problema (Maior complexidade operacional ao gerenciar múltiplos processos, portas e logs)|


**Para fechar** Uma loja com **3 desenvolvedores** deveria usar qual das duas versões? E uma com **300** ? Usem o placar como argumento. 

- **Com 3 desenvolvedores:** Deve usar a versão **Monolito**. Um time pequeno não possui braços suficientes para lidar com o elevado custo de operação, monitoramento distribuído e depuração de múltiplos processos. O monolito oferece alta produtividade, menor latência e consistência imediata dos dados com baixa complexidade.
- **Com 300 desenvolvedores:** Deve usar a versão **Microsserviços**. Com centenas de desenvolvedores trabalhando simultaneamente, o maior gargalo passa a ser o conflito de código, concorrência no deploy e acoplamento do monolito. Os microsserviços permitem a divisão em múltiplos times autônomos que realizam deploys independentes e isolam falhas, justificando a maior complexidade operacional e de rede.

Para recomeçar do zero: desliguem tudo (Ctrl+C nos terminais) e rodem `python resetar.py`. 
