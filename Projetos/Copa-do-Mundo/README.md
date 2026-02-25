# 🏆 História das Copas do Mundo | Power BI + HTML Integration

![Power BI](https://img.shields.io/badge/Power_BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white)
![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=for-the-badge&logo=css3&logoColor=white)
![DAX](https://img.shields.io/badge/DAX-Language-blue?style=for-the-badge)

> **"Quando o visual nativo atinge o limite, a engenharia entra em campo."**

Este projeto é um Dashboard interativo que narra a história completa de todas as Copas do Mundo (1930 - 2022). O diferencial técnico deste projeto é a **integração avançada de HTML & CSS dentro do Power BI**, permitindo criar visuais responsivos, estilização condicional complexa e uma navegação estilo "App" que seria impossível com componentes nativos.

---


## 🖼️ Visão Geral

O objetivo foi fugir do padrão "relatório corporativo" e criar uma experiência de usuário imersiva (App-Like Experience).

### Destaques Visuais (Front-End)
* **HTML Cards Dinâmicos:** A lista de partidas e artilheiros não usa visuais padrão. São códigos HTML gerados via DAX que permitem controle total de bordas (`border-radius`), sombras (`box-shadow`) e layout (`display: flex`).
* **Formatação Condicional CSS:** As barras de status (Vitória/Empate/Derrota) mudam de cor dinamicamente injetando CSS inline baseado no resultado do jogo.
* **Micro-Interações:** Tooltips personalizados em HTML que mostram detalhes dos gols ao passar o mouse sobre um artilheiro.

### Engenharia de Dados (Back-End)
* **ETL Complexo:** Unificação de duas fontes de dados distintas com tratamento de chaves inconsistentes (nomes de jogadores).
* **Normalização Histórica:** Tratamento de geopolítica (Alemanha Ocidental, URSS, Iugoslávia) para manter a continuidade dos dados históricos.
* **Regras de Negócio:** Adaptação da modelagem para formatos antigos de torneio (ex: Fase Final de 1930 e 1950 que eram quadrangulares e não mata-mata).

---

## 🛠️ Tecnologias Utilizadas

* **Microsoft Power BI:** Ferramenta principal de visualização e modelagem.
* **Linguagem DAX:** Usada não apenas para cálculos (Somas/Médias), mas para **manipulação de Strings** para gerar código HTML dinâmico.
* **HTML5 & CSS3:** Utilizados dentro do visual *HTML Content* para renderizar cartões, listas e KPIs.
* **Power Query (M):** Limpeza, saneamento e normalização das bases históricas.
* **Figma:** Prototipagem do background e layout.

---

## 🤝 Autor

**Gustavo Luz** *Data Analyst | Power BI Specialist*

## 📂 Fontes de Dados

Este projeto utiliza dados públicos disponíveis no Kaggle. Abaixo estão os detalhes dos datasets utilizados para análise estatística e visualização da FIFA World Cup.

### 1. FIFA World Cup (Historical)
* **Autor:** [Andre Becklas (abecklas)](https://www.kaggle.com/datasets/abecklas/fifa-world-cup)
* **Descrição:** Dados históricos abrangentes das Copas do Mundo de 1930 a 2014.
* **Ficheiros Utilizados:**
    * `WorldCupMatches.csv`: Resultados de todos os jogos individualmente.
    * `WorldCups.csv`: Resumo de cada torneio (Vencedor, Golos, Público, etc.).
    * `WorldCupPlayers.csv`: Informação sobre jogadores e alinhamentos.

### 2. FIFA Football World Cup (1930-2022)
* **Autor:** [Petro Ivaniuk (piterfm)](https://www.kaggle.com/datasets/piterfm/fifa-football-world-cup)
* **Descrição:** Dataset atualizado que inclui os torneios mais recentes, como o de 2018 e o do Qatar 2022.
* **Ficheiros Utilizados:**
    * `matches_1930_2022.csv`: Lista completa de jogos e resultados atualizados.
    * `world_cup.csv`: Informações gerais sobre as edições do torneio.
    * `fifa_ranking_2022-10-06.csv`: Rankings das seleções pré-2022.
