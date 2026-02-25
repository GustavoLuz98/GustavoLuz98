# 📚 Documentação Técnica das Medidas DAX

Este documento detalha as medidas DAX utilizadas no projeto **História das Copas**. O foco principal é a integração entre lógica de negócios (Power BI) e Front-End (HTML/CSS), permitindo a criação de visuais 100% customizáveis que superam as limitações nativas.

## 1. 🎨 Front-End: Visuais HTML & CSS Dinâmicos

Medidas responsáveis por gerar o código HTML injetado nos visuais customizados (HTML Content), controlando layout (Flexbox), cores condicionais e elementos gráficos (SVG).

### HTML_Card_Partida
Gera os cartões da lista de jogos (lateral direita). Utiliza lógica de **Flexbox** para alinhar bandeiras e placar. A borda lateral muda de cor dinamicamente (Verde/Amarelo/Vermelho) dependendo do saldo de gols, e as fases do torneio são traduzidas de Inglês para Português via `SWITCH`.
```dax
VAR TimeCasaOriginal = SELECTEDVALUE('WorldCupMatches'[Home Team Name])
VAR TimeForaOriginal = SELECTEDVALUE('WorldCupMatches'[Away Team Name])
VAR GolsCasa = SELECTEDVALUE('WorldCupMatches'[Home Team Goals])
VAR GolsFora = SELECTEDVALUE('WorldCupMatches'[Away Team Goals])
VAR FaseOriginal = SELECTEDVALUE('WorldCupMatches'[Stage])
VAR Ano = SELECTEDVALUE('WorldCupMatches'[Year]) 

-- 2. TRADUÇÃO E ABREVIAÇÃO DA FASE
VAR FasePT = 
    SWITCH(TRUE(),
        FaseOriginal = "Final", "Final",
        FaseOriginal = "Semi-finals", "Semi",
        FaseOriginal = "Quarter-finals", "Quart.",
        FaseOriginal = "Round of 16", "Oit.",
        FaseOriginal = "Third place", "3º Lug.",
        FaseOriginal = "Match for third place", "3º Lug.",
        FaseOriginal = "Group 6" && Ano = 1950, "Final",
        LEFT(FaseOriginal, 5) = "Group", "Grupos",
        FaseOriginal
    )

-- 3. Busca Nomes em Português e Bandeiras
VAR NomeCasaPT = CALCULATE(MAX('Dim_Paises'[Pais_PT]), 'Dim_Paises'[Pais] = TimeCasaOriginal, ALL('Dim_Paises'))
VAR NomeForaPT = CALCULATE(MAX('Dim_Paises'[Pais_PT]), 'Dim_Paises'[Pais] = TimeForaOriginal, ALL('Dim_Paises'))
VAR BandeiraCasa = CALCULATE(MAX('Dim_Paises'[URL_Bandeira]), 'Dim_Paises'[Pais] = TimeCasaOriginal, ALL('Dim_Paises'))
VAR BandeiraFora = CALCULATE(MAX('Dim_Paises'[URL_Bandeira]), 'Dim_Paises'[Pais] = TimeForaOriginal, ALL('Dim_Paises'))

-- Fallbacks (Segurança para não ficar em branco)
VAR TimeCasaDisplay = IF(ISBLANK(NomeCasaPT), TimeCasaOriginal, NomeCasaPT)
VAR TimeForaDisplay = IF(ISBLANK(NomeForaPT), TimeForaOriginal, NomeForaPT)
VAR ImgCasaFinal = IF(ISBLANK(BandeiraCasa), "[https://via.placeholder.com/50?text=](https://via.placeholder.com/50?text=)?", BandeiraCasa)
VAR ImgForaFinal = IF(ISBLANK(BandeiraFora), "[https://via.placeholder.com/50?text=](https://via.placeholder.com/50?text=)?", BandeiraFora)

-- 4. Definição Visual (Cores)
VAR CorPlacar = "#333333" 
VAR BordaCard = IF(GolsCasa > GolsFora, "border-left: 5px solid #2ecc71;", IF(GolsFora > GolsCasa, "border-right: 5px solid #2ecc71;", "border-left: 5px solid #f1c40f;"))

RETURN
-- HTML FINAL (Limpo, sem tooltip)
"<div style='background-color: white; margin-bottom: 10px; padding: 10px; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); display: flex; align-items: center; justify-content: space-between; font-family: Segoe UI, sans-serif; " & BordaCard & "'>" &

    -- Lado Esquerdo (Casa + Bandeira)
    "<div style='display: flex; align-items: center; width: 40%; justify-content: flex-start;'>" &
        "<img src='" & ImgCasaFinal & "' style='width: 40px; height: 40px; border-radius: 50%; border: 1px solid #eee; margin-right: 10px; object-fit: cover;'>" &
        "<span style='font-weight: 600; font-size: 14px; color: #444;'>" & TimeCasaDisplay & "</span>" &
    "</div>" &

    -- Centro (Placar + Ano/Fase)
    "<div style='width: 20%; text-align: center;'>" &
        "<div style='font-size: 24px; font-weight: 800; color: " & CorPlacar & ";'>" & GolsCasa & " - " & GolsFora & "</div>" &
        "<div style='font-size: 11px; color: #999; margin-top: -2px; font-weight: 600;'>" & 
            Ano & " - " & FasePT & 
        "</div>" &
    "</div>" &

    -- Lado Direito (Visitante + Bandeira)
    "<div style='display: flex; align-items: center; width: 40%; justify-content: flex-end;'>" &
        "<span style='font-weight: 600; font-size: 14px; color: #444; margin-right: 10px; text-align: right;'>" & TimeForaDisplay & "</span>" &
        "<img src='" & ImgForaFinal & "' style='width: 40px; height: 40px; border-radius: 50%; border: 1px solid #eee; object-fit: cover;'>" &
    "</div>" &

"</div>"
````

### HTML\_Painel\_Performance

Uma das medidas mais complexas do projeto. Ela gera um painel completo contendo:

1.  **Gráfico de Barras (Funil):** Renderizado via HTML/CSS (`width: %`) mostrando até onde a seleção chegou (Final, Semi, Quartas).
2.  **Gráficos de Rosca (Donut Charts):** Gerados via código **SVG** dentro do DAX para mostrar Aproveitamento e Pênaltis.
3.  **Estatísticas de Cartões:** Ícones e contadores de cartões amarelos e vermelhos.

<!-- end list -->

```dax
VAR PaisSel = SELECTEDVALUE('Dim_Paises'[Pais])

VAR TotalAmarelos = 
    CALCULATE(
        SUMX('WorldCupPlayers', LEN('WorldCupPlayers'[Event]) - LEN(SUBSTITUTE('WorldCupPlayers'[Event], "Y", ""))),
        TREATAS(VALUES('Dim_Paises'[Sigla]), 'WorldCupPlayers'[Team Initials])
    )

VAR TotalVermelhos = 
    CALCULATE(
        SUMX('WorldCupPlayers', 
            (LEN('WorldCupPlayers'[Event]) - LEN(SUBSTITUTE('WorldCupPlayers'[Event], "R", ""))) + 
            ((LEN('WorldCupPlayers'[Event]) - LEN(SUBSTITUTE('WorldCupPlayers'[Event], "SY", ""))) / 2) 
        ),
        TREATAS(VALUES('Dim_Paises'[Sigla]), 'WorldCupPlayers'[Team Initials])
    )

VAR TotalPenaltisConvertidos = 
    CALCULATE(
        SUMX('WorldCupPlayers', 
            VAR SemMP = SUBSTITUTE('WorldCupPlayers'[Event], "MP", "")
            RETURN LEN(SemMP) - LEN(SUBSTITUTE(SemMP, "P", ""))
        ),
        TREATAS(VALUES('Dim_Paises'[Sigla]), 'WorldCupPlayers'[Team Initials])
    )

VAR TotalPenaltisPerdidos = 
    CALCULATE(
        SUMX('WorldCupPlayers', (LEN('WorldCupPlayers'[Event]) - LEN(SUBSTITUTE('WorldCupPlayers'[Event], "MP", ""))) / 2),
        TREATAS(VALUES('Dim_Paises'[Sigla]), 'WorldCupPlayers'[Team Initials])
    )

VAR PercAproveitamento = [% Aproveitamento] 
VAR TotalPenaltis = TotalPenaltisConvertidos + TotalPenaltisPerdidos
VAR PercPenaltis = DIVIDE(TotalPenaltisConvertidos, TotalPenaltis, 0)

-- ==========================================================
-- 2. CÁLCULOS DO MATA-MATA
-- ==========================================================

VAR QtdFinal = 
    CALCULATE(
        DISTINCTCOUNT('WorldCupMatches'[Year]),
        FILTER(
            'WorldCupMatches',
            ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) &&
            (
                'WorldCupMatches'[Stage] = "Final" || 
                ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)
            )
        )
    )

VAR QtdSemi = 
    CALCULATE(
        DISTINCTCOUNT('WorldCupMatches'[Year]),
        FILTER(
            'WorldCupMatches',
            ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) &&
            (
                'WorldCupMatches'[Stage] IN {"Final", "Semi-finals", "Third place", "Match for third place"} || 
                ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)
            )
        )
    )

VAR QtdQuartas = 
    CALCULATE(
        DISTINCTCOUNT('WorldCupMatches'[Year]),
        FILTER(
            'WorldCupMatches',
            ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) &&
            (
                'WorldCupMatches'[Stage] IN {"Final", "Semi-finals", "Third place", "Match for third place", "Quarter-finals"} || 
                ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)
            )
        )
    )

VAR QtdOitavas = 
    CALCULATE(
        DISTINCTCOUNT('WorldCupMatches'[Year]),
        FILTER(
            'WorldCupMatches',
            ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) &&
            (
                'WorldCupMatches'[Stage] IN {"Final", "Semi-finals", "Third place", "Match for third place", "Quarter-finals", "Round of 16"} || 
                ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)
            )
        )
    )

-- Escala e Larguras
VAR MaxBarra = MAXX({QtdFinal, QtdSemi, QtdQuartas, QtdOitavas}, [Value])
VAR Denominador = IF(MaxBarra = 0, 1, MaxBarra)
VAR WidthFinal = INT(DIVIDE(QtdFinal, Denominador, 0) * 100)
VAR WidthSemi = INT(DIVIDE(QtdSemi, Denominador, 0) * 100)
VAR WidthQuartas = INT(DIVIDE(QtdQuartas, Denominador, 0) * 100)
VAR WidthOitavas = INT(DIVIDE(QtdOitavas, Denominador, 0) * 100)

-- ==========================================================
-- 3. LISTAS DE ANOS (TOOLTIPS)
-- ==========================================================
-- ... (Mesma lógica de listas de antes, mantida para funcionar o tooltip) ...
VAR TabelaFinal = FILTER('WorldCupMatches', ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) && ('WorldCupMatches'[Stage] = "Final" || ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)))
VAR ListFinal = CALCULATE(CONCATENATEX(VALUES('WorldCupMatches'[Year]), 'WorldCupMatches'[Year], ", ", 'WorldCupMatches'[Year], ASC), TabelaFinal)

VAR TabelaSemi = FILTER('WorldCupMatches', ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) && ('WorldCupMatches'[Stage] IN {"Final", "Semi-finals", "Third place", "Match for third place"} || ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)))
VAR ListSemi = CALCULATE(CONCATENATEX(VALUES('WorldCupMatches'[Year]), 'WorldCupMatches'[Year], ", ", 'WorldCupMatches'[Year], ASC), TabelaSemi)

VAR TabelaQuartas = FILTER('WorldCupMatches', ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) && ('WorldCupMatches'[Stage] IN {"Final", "Semi-finals", "Third place", "Match for third place", "Quarter-finals"} || ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)))
VAR ListQuartas = CALCULATE(CONCATENATEX(VALUES('WorldCupMatches'[Year]), 'WorldCupMatches'[Year], ", ", 'WorldCupMatches'[Year], ASC), TabelaQuartas)

VAR TabelaOitavas = FILTER('WorldCupMatches', ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) && ('WorldCupMatches'[Stage] IN {"Final", "Semi-finals", "Third place", "Match for third place", "Quarter-finals", "Round of 16"} || ('WorldCupMatches'[Stage] = "Group 6" && 'WorldCupMatches'[Year] = 1950)))
VAR ListOitavas = CALCULATE(CONCATENATEX(VALUES('WorldCupMatches'[Year]), 'WorldCupMatches'[Year], ", ", 'WorldCupMatches'[Year], ASC), TabelaOitavas)


-- ==========================================================
-- 4. FUNÇÕES DE DESENHO (SVG)
-- ==========================================================
VAR CorAp = IF(PercAproveitamento >= 0.5, "#2ecc71", "#e74c3c")
VAR DashAp = (2 * 3.14 * 40) * PercAproveitamento
VAR SVGAproveitamento = 
    "<svg viewBox='0 0 100 100' xmlns='[http://www.w3.org/2000/svg](http://www.w3.org/2000/svg)' width='90' height='90'>" &
      "<circle cx='50' cy='50' r='40' fill='none' stroke='#f5f5f5' stroke-width='8' />" &
      "<circle cx='50' cy='50' r='40' fill='none' stroke='" & CorAp & "' stroke-width='8' stroke-dasharray='" & DashAp & " 251' transform='rotate(-90 50 50)' stroke-linecap='round' />" &
      "<text x='50' y='55' font-family='Segoe UI' font-weight='800' font-size='22' text-anchor='middle' fill='#333'>" & FORMAT(PercAproveitamento, "0%") & "</text>" &
    "</svg>"

VAR CorPen = IF(PercPenaltis >= 0.5, "#2ecc71", "#e74c3c")
VAR DashPen = (2 * 3.14 * 40) * PercPenaltis
VAR SVGPenaltis = 
    "<svg viewBox='0 0 100 100' xmlns='[http://www.w3.org/2000/svg](http://www.w3.org/2000/svg)' width='90' height='90'>" &
      "<circle cx='50' cy='50' r='40' fill='none' stroke='#f5f5f5' stroke-width='8' />" &
      "<circle cx='50' cy='50' r='40' fill='none' stroke='" & CorPen & "' stroke-width='8' stroke-dasharray='" & DashPen & " 251' transform='rotate(-90 50 50)' stroke-linecap='round' />" &
      "<text x='50' y='48' font-family='Segoe UI' font-weight='800' font-size='20' text-anchor='middle' fill='#333'>" & FORMAT(PercPenaltis, "0%") & "</text>" &
      "<text x='50' y='68' font-family='Segoe UI' font-size='12' text-anchor='middle' fill='#888'>Conv.</text>" &
    "</svg>"

-- ==========================================================
-- 5. LAYOUT HTML FINAL (COM CORREÇÃO DE ZEROS)
-- ==========================================================
RETURN
"<div style='background-color: white; padding: 20px 10px; border-radius: 15px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); font-family: Segoe UI, sans-serif; display: flex; flex-direction: column; justify-content: flex-start; align-items: center; height: 100%; width: 100%; gap: 10px; cursor: default;'>" &

    -- >>> PARTE SUPERIOR (GRÁFICO DE MATA-MATA - FUNIL) <<<
    "<div style='width: 90%; display: flex; flex-direction: column; gap: 8px;'>" &
        "<div style='font-size: 12px; font-weight: 800; color: #333; margin-bottom: 2px; display: flex; align-items: center;'><span style='margin-right:5px;'>🏆</span> Fases Alcançadas</div>" &
        
        -- FINAL (Ajuste: Se Qtd=0, display:none)
        "<div title='Anos: " & ListFinal & "' style='display: flex; align-items: center; width: 100%; font-size: 11px; cursor: help;'>" &
            "<div style='width: 50px; color: #666; font-weight: 600;'>Final</div>" &
            "<div style='flex: 1; background-color: #f9f9f9; height: 12px; border-radius: 4px; overflow: hidden;'>" &
                "<div style='" & IF(QtdFinal = 0, "display: none;", "width: " & WidthFinal & "%;") & " background-color: #f1c40f; height: 100%; border-radius: 4px;'></div>" &
            "</div>" &
            "<div style='width: 25px; text-align: right; font-weight: 700; color: #444;'>" & QtdFinal & "</div>" &
        "</div>" &

        -- SEMI (Ajuste: Se Qtd=0, display:none)
        "<div title='Anos: " & ListSemi & "' style='display: flex; align-items: center; width: 100%; font-size: 11px; cursor: help;'>" &
            "<div style='width: 50px; color: #666;'>Semi</div>" &
            "<div style='flex: 1; background-color: #f9f9f9; height: 12px; border-radius: 4px; overflow: hidden;'>" &
                "<div style='" & IF(QtdSemi = 0, "display: none;", "width: " & WidthSemi & "%;") & " background-color: #3498db; height: 100%; border-radius: 4px;'></div>" &
            "</div>" &
            "<div style='width: 25px; text-align: right; font-weight: 700; color: #444;'>" & QtdSemi & "</div>" &
        "</div>" &

        -- QUARTAS (Ajuste: Se Qtd=0, display:none)
        "<div title='Anos: " & ListQuartas & "' style='display: flex; align-items: center; width: 100%; font-size: 11px; cursor: help;'>" &
            "<div style='width: 50px; color: #666;'>Quartas</div>" &
            "<div style='flex: 1; background-color: #f9f9f9; height: 12px; border-radius: 4px; overflow: hidden;'>" &
                "<div style='" & IF(QtdQuartas = 0, "display: none;", "width: " & WidthQuartas & "%;") & " background-color: #e67e22; height: 100%; border-radius: 4px;'></div>" &
            "</div>" &
            "<div style='width: 25px; text-align: right; font-weight: 700; color: #444;'>" & QtdQuartas & "</div>" &
        "</div>" &

        -- OITAVAS (Ajuste: Se Qtd=0, display:none)
        "<div title='Anos: " & ListOitavas & "' style='display: flex; align-items: center; width: 100%; font-size: 11px; cursor: help;'>" &
            "<div style='width: 50px; color: #666;'>Oitavas</div>" &
            "<div style='flex: 1; background-color: #f9f9f9; height: 12px; border-radius: 4px; overflow: hidden;'>" &
                "<div style='" & IF(QtdOitavas = 0, "display: none;", "width: " & WidthOitavas & "%;") & " background-color: #2ecc71; height: 100%; border-radius: 4px;'></div>" &
            "</div>" &
            "<div style='width: 25px; text-align: right; font-weight: 700; color: #444;'>" & QtdOitavas & "</div>" &
        "</div>" &

    "</div>" &

    -- >>> DIVISOR PRINCIPAL <<<
    "<div style='width: 90%; height: 2px; background-color: #ccc; margin: 15px 0;'></div>" &

    -- >>> PARTE INFERIOR (ESTATÍSTICAS) <<<
    -- 1. APROVEITAMENTO
    "<div title='Vitórias, Empates e Derrotas ponderados' style='display: flex; flex-direction: column; align-items: center; justify-content: center;'>" &
        SVGAproveitamento &
        "<div style='font-size: 11px; color: #666; margin-top: 5px; font-weight: 700; text-transform: uppercase;'>Aproveitamento</div>" &
    "</div>" &

    -- DIVISOR 1
    "<div style='width: 70%; height: 2px; background-color: #e0e0e0; border-radius: 2px; margin: 5px 0;'></div>" &

    -- 2. CARTÕES
    "<div title='Total acumulado em todas as Copas' style='display: flex; flex-direction: column; align-items: center; justify-content: center;'>" &
        "<div style='display: flex; gap: 15px; margin-bottom: 5px;'>" &
            "<div style='display: flex; align-items: center;'>" &
                "<div style='width: 12px; height: 16px; background-color: #f1c40f; border-radius: 2px; margin-right: 6px;'></div>" & 
                "<span style='font-weight: 800; font-size: 18px; color: #444;'>" & TotalAmarelos & "</span>" &
            "</div>" &
            "<div style='display: flex; align-items: center;'>" &
                "<div style='width: 12px; height: 16px; background-color: #e74c3c; border-radius: 2px; margin-right: 6px;'></div>" & 
                "<span style='font-weight: 800; font-size: 18px; color: #444;'>" & TotalVermelhos & "</span>" &
            "</div>" &
        "</div>" &
        "<div style='font-size: 11px; color: #666; font-weight: 700; text-transform: uppercase;'>Cartões</div>" &
    "</div>" &

    -- DIVISOR 2
    "<div style='width: 70%; height: 2px; background-color: #e0e0e0; border-radius: 2px; margin: 5px 0;'></div>" &

    -- 3. PÊNALTIS
    "<div title='Convertidos: " & TotalPenaltisConvertidos & " / Perdidos: " & TotalPenaltisPerdidos & "' style='display: flex; flex-direction: column; align-items: center; justify-content: center; cursor: help;'>" &
        SVGPenaltis &
        "<div style='font-size: 11px; color: #666; margin-top: 5px; font-weight: 700; text-transform: uppercase;'>Pênaltis</div>" &
    "</div>" &

"</div>"
```

### HTML\_Ranking\_Global

Gera a tabela completa de medalhas (ranking geral) com bandeiras e colunas para Ouro, Prata e Bronze, destacando o background das seleções campeãs.

```dax
VAR TabelaMedalhas = 
    ADDCOLUMNS(
        VALUES('Dim_Paises'[Pais]), 
        "NomePT", CALCULATE(MAX('Dim_Paises'[Pais_PT])),
        "Flag", CALCULATE(MAX('Dim_Paises'[URL_Bandeira])),
        "Ouro", VAR PK = 'Dim_Paises'[Pais] RETURN CALCULATE(COUNTROWS('WorldCups'), 'WorldCups'[Winner] = PK),
        "Prata", VAR PK = 'Dim_Paises'[Pais] RETURN CALCULATE(COUNTROWS('WorldCups'), 'WorldCups'[Runners-Up] = PK),
        "Bronze", VAR PK = 'Dim_Paises'[Pais] RETURN CALCULATE(COUNTROWS('WorldCups'), 'WorldCups'[Third] = PK)
    )
VAR TabelaClassificada = TOPN(20, FILTER(TabelaMedalhas, [Ouro]>0||[Prata]>0||[Bronze]>0), [Ouro]*10000+[Prata]*100+[Bronze], DESC)

VAR LinhasHTML = 
    CONCATENATEX(
        TabelaClassificada,
        VAR CorFundo = IF([Ouro] > 0, "#fffcf2", "white")
        VAR Img = IF(ISBLANK([Flag]), "[https://via.placeholder.com/30](https://via.placeholder.com/30)", [Flag])
        VAR NomeDisplay = IF(ISBLANK([NomePT]), 'Dim_Paises'[Pais], [NomePT])
        RETURN
        "<div style='display: flex; align-items: center; padding: 8px 10px; border-bottom: 1px solid #f0f0f0; background-color: " & CorFundo & "; flex-shrink: 0;'>" &
            "<div style='width: 40%; display: flex; align-items: center;'>" &
                "<img src='" & Img & "' style='width: 25px; height: 25px; border-radius: 50%; border: 1px solid #ddd; margin-right: 8px; object-fit: cover;'>" &
                "<span style='font-weight: 600; color: #444; font-size: 13px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;'>" & NomeDisplay & "</span>" &
            "</div>" &
            "<div style='width: 20%; text-align: center; font-weight: 800; color: #f1c40f; font-size: 14px;'>" & [Ouro] & "</div>" &
            "<div style='width: 20%; text-align: center; font-weight: 700; color: #95a5a6; font-size: 14px;'>" & [Prata] & "</div>" &
            "<div style='width: 20%; text-align: center; font-weight: 700; color: #d35400; font-size: 14px;'>" & [Bronze] & "</div>" &
        "</div>",
        "", [Ouro]*10000+[Prata]*100+[Bronze], DESC
    )

RETURN
"<div style='background-color: white; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); font-family: Segoe UI, sans-serif; height: 100%; display: flex; flex-direction: column; overflow: hidden;'>" &
    
    -- CABEÇALHO PADRONIZADO (45px Altura)
    "<div style='background-color: #333; color: white; height: 45px; padding: 0 15px; display: flex; align-items: center; flex-shrink: 0; box-shadow: 0 2px 4px rgba(0,0,0,0.1); z-index: 10;'>" &
        "<span style='font-size: 18px; margin-right: 10px;'>🏆</span>" &
        "<span style='font-size: 14px; font-weight: 700; letter-spacing: 0.5px; text-transform: uppercase;'>Quadro de Medalhas</span>" &
    "</div>" &

    -- SUB-CABEÇALHO FIXO (Cinza)
    "<div style='display: flex; padding: 8px 10px; background-color: #f8f9fa; color: #666; font-weight: 700; font-size: 11px; text-transform: uppercase; border-bottom: 1px solid #eee; flex-shrink: 0;'>" &
        "<div style='width: 40%;'>Seleção</div>" &
        "<div style='width: 20%; text-align: center;'>🥇</div>" &
        "<div style='width: 20%; text-align: center;'>🥈</div>" &
        "<div style='width: 20%; text-align: center;'>🥉</div>" &
    "</div>" &

    -- LISTA MÓVEL
    "<div style='overflow-y: auto; flex-grow: 1; scrollbar-width: thin;'>" & LinhasHTML & "</div>" &
"</div>"
```

### HTML\_Top5\_Artilheiros / HTML\_Top5\_Artilheiros\_2

Cria uma lista rolável dos artilheiros. O diferencial técnico aqui é o **Tooltip Customizado via HTML**: ao passar o mouse sobre o jogador, uma lista detalhada aparece mostrando contra quem ele marcou os gols.

```dax
VAR TabelaJogadoresFiltrada = 
    CALCULATETABLE(
        SUMMARIZE('WorldCupPlayers', 'WorldCupPlayers'[Player Name], 'WorldCupPlayers'[Team Initials]),
        TREATAS(VALUES('Dim_Paises'[Sigla]), 'WorldCupPlayers'[Team Initials])
    )

-- 2. Calcula o Top 100
VAR TabelaTop5 = 
    TOPN(
        100, 
        TabelaJogadoresFiltrada, 
        [Contagem_Gols_Events], 
        DESC
    )

-- 3. Monta o HTML
VAR ListaHTML = 
    CONCATENATEX(
        TabelaTop5,
        VAR Nome = 'WorldCupPlayers'[Player Name]
        VAR Gols = [Contagem_Gols_Events]
        VAR SiglaTime = 'WorldCupPlayers'[Team Initials]
        
        -- Busca Bandeira do Jogador
        VAR BandeiraURL = 
            CALCULATE(
                MAX('Dim_Paises'[URL_Bandeira]),
                'Dim_Paises'[Sigla] = SiglaTime,
                REMOVEFILTERS('Dim_Paises')
            )
        VAR ImgFinal = IF(ISBLANK(BandeiraURL), "[https://via.placeholder.com/30](https://via.placeholder.com/30)", BandeiraURL)
        
        -- >>> CORREÇÃO DO TOOLTIP (NOMES EM PORTUGUÊS) <<<
        VAR TabelaComGols = 
            ADDCOLUMNS(
                FILTER('WorldCupPlayers', 'WorldCupPlayers'[Player Name] = Nome),
                "Adversario",
                VAR MatchID = 'WorldCupPlayers'[MatchID]
                -- Pega nomes originais (Inglês) da partida
                VAR HomeName = CALCULATE(MAX('WorldCupMatches'[Home Team Name]), 'WorldCupMatches'[MatchID] = MatchID)
                VAR AwayName = CALCULATE(MAX('WorldCupMatches'[Away Team Name]), 'WorldCupMatches'[MatchID] = MatchID)
                
                -- Descobre a Sigla do time da Casa para saber quem é quem
                VAR SiglaCasa = CALCULATE(MAX('Dim_Paises'[Sigla]), 'Dim_Paises'[Pais] = HomeName, REMOVEFILTERS('Dim_Paises'))
                
                -- Identifica o adversário em Inglês (Chave)
                VAR AdversarioIngles = IF(SiglaCasa = SiglaTime, AwayName, HomeName)
                
                -- Busca a tradução na coluna Pais_PT
                VAR AdversarioPT = 
                    CALCULATE(
                        MAX('Dim_Paises'[Pais_PT]), 
                        'Dim_Paises'[Pais] = AdversarioIngles, 
                        REMOVEFILTERS('Dim_Paises') -- Remove filtros para poder achar qualquer país adversário
                    )
                
                -- Retorna o nome em PT (ou o Inglês se não achar tradução)
                RETURN IF(ISBLANK(AdversarioPT), AdversarioIngles, AdversarioPT),
                
                "GolsNaLinha", [Contagem_Gols_Events]
            )
        
        -- Agrupa e Soma (Tooltip Final)
        VAR ListaAdversarios = 
            CONCATENATEX(
                GROUPBY(
                    FILTER(TabelaComGols, [GolsNaLinha] > 0), 
                    [Adversario],
                    "QtdGols", SUMX(CURRENTGROUP(), [GolsNaLinha])
                ),
                [Adversario] & ": " & [QtdGols],
                " | ",
                [QtdGols], DESC
            )

        RETURN
        "<div title='" & ListaAdversarios & "' style='display: flex; align-items: center; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #f0f0f0; cursor: help; transition: background-color 0.2s;'>" &
            "<div style='display: flex; align-items: center;'>" &
                "<img src='" & ImgFinal & "' style='width: 30px; height: 30px; border-radius: 50%; border: 1px solid #ddd; margin-right: 12px; object-fit: cover;'>" &
                "<div style='font-size: 14px; font-weight: 600; color: #444;'>" & Nome & "</div>" &
            "</div>" &
            "<div style='background-color: #333; color: white; padding: 4px 10px; border-radius: 12px; font-size: 12px; font-weight: bold; min-width: 30px; text-align: center;'>" & 
                Gols & " ⚽" & 
            "</div>" &
        "</div>",
        
        "", 
        
        [Contagem_Gols_Events], 
        DESC
    )

RETURN
"<div style='background-color: white; padding: 15px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); font-family: Segoe UI, sans-serif;'>" &
    "<div style='font-size: 16px; font-weight: 800; color: #333; margin-bottom: 15px; display: flex; align-items: center; border-bottom: 2px solid #eee; padding-bottom: 10px;'>" &
        "<span style='font-size: 20px; margin-right: 8px;'>👟</span>Artilheiros" &
    "</div>" &
    ListaHTML &
"</div>"
```

### HTML\_Card\_Resumo\_Copa

Cartão de destaque que se adapta: se nenhum ano for filtrado, mostra o resumo histórico. Se um ano for filtrado, mostra os dados daquela edição (Campeão, Público, Gols).

```dax
VAR AnoSelecionado = SELECTEDVALUE('WorldCups'[Year])
VAR TemFiltro = NOT(ISBLANK(AnoSelecionado))

-- (Variáveis de cálculo)
VAR TotalCopas = COUNTROWS('WorldCups')
VAR TotalGolsHistoria = SUM('WorldCups'[GoalsScored])
VAR TotalPublicoHistoria = SUM('WorldCups'[Attendance]) -- NOVO: Público Total

VAR PaisSedeOriginal = SELECTEDVALUE('WorldCups'[Country])
VAR CampeaoOriginal = SELECTEDVALUE('WorldCups'[Winner])
VAR ViceOriginal = SELECTEDVALUE('WorldCups'[Runners-Up])
VAR GolsAno = SELECTEDVALUE('WorldCups'[GoalsScored])
VAR PartidasAno = SELECTEDVALUE('WorldCups'[MatchesPlayed])
VAR PublicoAno = SELECTEDVALUE('WorldCups'[Attendance]) -- NOVO: Público do Ano
VAR MediaGols = DIVIDE(GolsAno, PartidasAno, 0)

-- Traduções
VAR NomeSedePT = CALCULATE(MAX('Dim_Paises'[Pais_PT]), 'Dim_Paises'[Pais] = PaisSedeOriginal, ALL('Dim_Paises'))
VAR NomeCampeaoPT = CALCULATE(MAX('Dim_Paises'[Pais_PT]), 'Dim_Paises'[Pais] = CampeaoOriginal, ALL('Dim_Paises'))
VAR NomeVicePT = CALCULATE(MAX('Dim_Paises'[Pais_PT]), 'Dim_Paises'[Pais] = ViceOriginal, ALL('Dim_Paises'))
VAR SedeDisplay = IF(ISBLANK(NomeSedePT), PaisSedeOriginal, NomeSedePT)
VAR CampeaoDisplay = IF(ISBLANK(NomeCampeaoPT), CampeaoOriginal, NomeCampeaoPT)
VAR ViceDisplay = IF(ISBLANK(NomeVicePT), ViceOriginal, NomeVicePT)
VAR BandeiraCampeao = CALCULATE(MAX('Dim_Paises'[URL_Bandeira]), 'Dim_Paises'[Pais] = CampeaoOriginal, ALL('Dim_Paises'))
VAR ImgCampeao = IF(ISBLANK(BandeiraCampeao), "[https://via.placeholder.com/60?text=](https://via.placeholder.com/60?text=)?", BandeiraCampeao)

RETURN
IF(TemFiltro,
    -- >>> COM FILTRO (DETALHE ANO) <<<
    "<div style='background-color: white; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); font-family: Segoe UI, sans-serif; height: 100%; display: flex; flex-direction: column; overflow: hidden;'>" &
        
        -- CABEÇALHO PADRONIZADO
        "<div style='background-color: #333; color: white; height: 45px; padding: 0 15px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; box-shadow: 0 2px 4px rgba(0,0,0,0.1); z-index: 10;'>" &
             "<span style='font-size: 14px; font-weight: 700; letter-spacing: 0.5px; text-transform: uppercase;'>" & AnoSelecionado & " • " & SedeDisplay & "</span>" &
        "</div>" &

        -- CONTEÚDO MÓVEL
        "<div style='padding: 20px; display: flex; flex-direction: column; justify-content: space-between; flex-grow: 1; overflow-y: auto;'>" &
             -- Bloco Campeão
             "<div style='background: linear-gradient(135deg, #fdfbfb 0%, #ebedee 100%); border-radius: 12px; padding: 15px; display: flex; align-items: center; justify-content: space-between; border: 1px solid #ddd; margin-bottom: 20px;'>" &
                "<div style='display: flex; flex-direction: column;'>" &
                    "<span style='font-size: 11px; font-weight: 700; color: #f39c12; text-transform: uppercase; letter-spacing: 1px;'>Grande Campeão</span>" &
                    "<span style='font-size: 24px; font-weight: 900; color: #2c3e50;'>" & CampeaoDisplay & "</span>" &
                    "<span style='font-size: 12px; color: #7f8c8d;'>Vice: " & ViceDisplay & "</span>" &
                "</div>" &
                "<img src='" & ImgCampeao & "' style='width: 70px; height: 70px; border-radius: 50%; border: 3px solid white; box-shadow: 0 4px 6px rgba(0,0,0,0.15); object-fit: cover;'>" &
            "</div>" &
            
            -- Stats Grid (Agora com 4 colunas: Gols, Média, Jogos, Público)
            "<div style='display: flex; justify-content: space-around; padding-top: 10px; align-items: center;'>" &
                -- Gols
                "<div style='text-align: center;'><div style='font-size: 18px; font-weight: 800; color: #333;'>" & GolsAno & "</div><div style='font-size: 10px; color: #888; text-transform: uppercase;'>Gols</div></div>" &
                "<div style='width: 1px; height: 30px; background: #eee;'></div>" &
                -- Média
                "<div style='text-align: center;'><div style='font-size: 18px; font-weight: 800; color: #333;'>" & FORMAT(MediaGols, "0.0") & "</div><div style='font-size: 10px; color: #888; text-transform: uppercase;'>Média</div></div>" &
                "<div style='width: 1px; height: 30px; background: #eee;'></div>" &
                -- Jogos
                "<div style='text-align: center;'><div style='font-size: 18px; font-weight: 800; color: #333;'>" & FORMAT(PartidasAno, "#") & "</div><div style='font-size: 10px; color: #888; text-transform: uppercase;'>Jogos</div></div>" &
                "<div style='width: 1px; height: 30px; background: #eee;'></div>" &
                -- Público (NOVO) - Formata como milhar (ex: 3.404.252) ou Milhão compacto se preferir
                "<div style='text-align: center;'><div style='font-size: 18px; font-weight: 800; color: #333;'>" & FORMAT(PublicoAno, "#,,.0M") & "</div><div style='font-size: 10px; color: #888; text-transform: uppercase;'>Público</div></div>" &
            "</div>" &
        "</div>" &
    "</div>",

    -- >>> SEM FILTRO (GERAL HISTÓRICO) <<<
    "<div style='background-color: white; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); font-family: Segoe UI, sans-serif; height: 100%; display: flex; flex-direction: column; overflow: hidden;'>" &
        
        -- CABEÇALHO PADRONIZADO
        "<div style='background-color: #333; color: white; height: 45px; padding: 0 15px; display: flex; align-items: center; justify-content: center; flex-shrink: 0; box-shadow: 0 2px 4px rgba(0,0,0,0.1); z-index: 10;'>" &
             "<span style='font-size: 18px; margin-right: 10px;'>🌍</span>" &
             "<span style='font-size: 14px; font-weight: 700; letter-spacing: 0.5px; text-transform: uppercase;'>História da Copa</span>" &
        "</div>" &

        -- CONTEÚDO MÓVEL
        "<div style='padding: 20px; display: flex; flex-direction: column; justify-content: center; align-items: center; flex-grow: 1; text-align: center; gap: 20px; overflow-y: auto;'>" &
            "<div style='font-size: 14px; color: #666;'>Selecione um ano acima para ver os detalhes da edição.</div>" &
            "<div style='display: flex; gap: 20px; justify-content: center; width: 100%;'>" &
                "<div><div style='font-weight: 900; font-size: 22px; color: #2c3e50;'>" & TotalCopas & "</div><div style='font-size: 10px; color: #999; text-transform: uppercase;'>Edições</div></div>" &
                "<div><div style='font-weight: 900; font-size: 22px; color: #2c3e50;'>" & FORMAT(TotalGolsHistoria, "#,##0") & "</div><div style='font-size: 10px; color: #999; text-transform: uppercase;'>Gols</div></div>" &
                -- NOVO: Público Total em Milhões (ex: 40.5 M)
                "<div><div style='font-weight: 900; font-size: 22px; color: #2c3e50;'>" & FORMAT(TotalPublicoHistoria, "#,0,,.0 M") & "</div><div style='font-size: 10px; color: #999; text-transform: uppercase;'>Público Total</div></div>" &
            "</div>" &
        "</div>" &
    "</div>"
)
```

### HTML\_Card\_Podio

Mostra medalhas de Ouro, Prata e Bronze do país selecionado, incluindo um tooltip nos ícones que lista os anos das conquistas.

```dax
VAR PaisSel = SELECTEDVALUE('Dim_Paises'[Pais])
VAR QtdOuro = CALCULATE(COUNTROWS('WorldCups'), 'WorldCups'[Winner] = PaisSel)
VAR QtdPrata = CALCULATE(COUNTROWS('WorldCups'), 'WorldCups'[Runners-Up] = PaisSel)
VAR QtdBronze = CALCULATE(COUNTROWS('WorldCups'), 'WorldCups'[Third] = PaisSel)

VAR ListOuro = CALCULATE(CONCATENATEX(VALUES('WorldCups'[Year]), 'WorldCups'[Year], ", ", 'WorldCups'[Year], ASC), 'WorldCups'[Winner] = PaisSel)
VAR ListPrata = CALCULATE(CONCATENATEX(VALUES('WorldCups'[Year]), 'WorldCups'[Year], ", ", 'WorldCups'[Year], ASC), 'WorldCups'[Runners-Up] = PaisSel)
VAR ListBronze = CALCULATE(CONCATENATEX(VALUES('WorldCups'[Year]), 'WorldCups'[Year], ", ", 'WorldCups'[Year], ASC), 'WorldCups'[Third] = PaisSel)

VAR ShowOuro = IF(ISBLANK(QtdOuro), 0, QtdOuro)
VAR ShowPrata = IF(ISBLANK(QtdPrata), 0, QtdPrata)
VAR ShowBronze = IF(ISBLANK(QtdBronze), 0, QtdBronze)

RETURN
"<div style='background-color: white; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); font-family: Segoe UI, sans-serif; height: 100%; display: flex; flex-direction: column; overflow: hidden;'>" &

    -- CABEÇALHO PADRONIZADO (45px Altura - Igual aos outros)
    "<div style='background-color: #333; color: white; height: 45px; padding: 0 15px; display: flex; align-items: center; flex-shrink: 0; box-shadow: 0 2px 4px rgba(0,0,0,0.1); z-index: 10;'>" &
        "<span style='font-size: 18px; margin-right: 10px;'>🏅</span>" &
        "<span style='font-size: 14px; font-weight: 700; letter-spacing: 0.5px; text-transform: uppercase;'>Histórico de Pódio</span>" &
    "</div>" &

    -- CONTEÚDO (Centralizado Verticalmente)
    "<div style='padding: 10px; display: flex; justify-content: space-around; align-items: center; flex-grow: 1;'>" &
        
        -- OURO
        "<div title='Anos: " & ListOuro & "' style='display: flex; flex-direction: column; align-items: center; justify-content: center; cursor: help;'>" &
            "<div style='font-size: 36px; line-height: 1; filter: drop-shadow(0 2px 2px rgba(0,0,0,0.1));'>🥇</div>" &
            -- Texto centralizado com margem igual em cima e embaixo (8px)
            "<div style='font-size: 11px; color: #999; font-weight: 700; text-transform: uppercase; margin: 8px 0;'>Campeão</div>" &
            "<div style='font-size: 26px; font-weight: 900; color: #f1c40f; line-height: 1;'>" & ShowOuro & "</div>" &
        "</div>" &
        
        -- DIVISOR
        "<div style='width: 1px; height: 50px; background-color: #eee;'></div>" &
        
        -- PRATA
        "<div title='Anos: " & ListPrata & "' style='display: flex; flex-direction: column; align-items: center; justify-content: center; cursor: help;'>" &
            "<div style='font-size: 36px; line-height: 1; filter: drop-shadow(0 2px 2px rgba(0,0,0,0.1));'>🥈</div>" &
            "<div style='font-size: 11px; color: #999; font-weight: 700; text-transform: uppercase; margin: 8px 0;'>Vice</div>" &
            "<div style='font-size: 26px; font-weight: 900; color: #95a5a6; line-height: 1;'>" & ShowPrata & "</div>" &
        "</div>" &
        
        -- DIVISOR
        "<div style='width: 1px; height: 50px; background-color: #eee;'></div>" &
        
        -- BRONZE
        "<div title='Anos: " & ListBronze & "' style='display: flex; flex-direction: column; align-items: center; justify-content: center; cursor: help;'>" &
            "<div style='font-size: 36px; line-height: 1; filter: drop-shadow(0 2px 2px rgba(0,0,0,0.1));'>🥉</div>" &
            "<div style='font-size: 11px; color: #999; font-weight: 700; text-transform: uppercase; margin: 8px 0;'>3º Lugar</div>" &
            "<div style='font-size: 26px; font-weight: 900; color: #d35400; line-height: 1;'>" & ShowBronze & "</div>" &
        "</div>" &

    "</div>" &
"</div>"
```

### HTML\_KPI\_Status

Cria as "pílulas" de status (Vitórias, Empates, Derrotas) com cores condicionais e sombras suaves.

```dax
VAR V = [Vitorias]
VAR E = [Empates]
VAR D = [Derrotas]

RETURN
-- Container Principal com justify-content: center para centralizar tudo
"<div style='display: flex; gap: 15px; justify-content: center; width: 100%; font-family: Segoe UI, sans-serif;'>" &
    
    -- Pílula Verde (Vitórias)
    "<div style='background-color: #e8f5e9; color: #2e7d32; padding: 6px 18px; border-radius: 20px; font-weight: bold; border: 1px solid #c8e6c9; text-align: center; box-shadow: 0 2px 4px rgba(0,0,0,0.05);'>" &
        "Vitórias: " & V &
    "</div>" &

    -- Pílula Amarela (Empates)
    "<div style='background-color: #fffbe6; color: #b7950b; padding: 6px 18px; border-radius: 20px; font-weight: bold; border: 1px solid #f1c40f; text-align: center; box-shadow: 0 2px 4px rgba(0,0,0,0.05);'>" &
        "Empates: " & E &
    "</div>" &

    -- Pílula Vermelha (Derrotas)
    "<div style='background-color: #ffebee; color: #c62828; padding: 6px 18px; border-radius: 20px; font-weight: bold; border: 1px solid #ffcdd2; text-align: center; box-shadow: 0 2px 4px rgba(0,0,0,0.05);'>" &
        "Derrotas: " & D &
    "</div>" &

"</div>"
```

### HTML\_Stats\_Gols

Painel de 3 colunas para Gols Pró, Saldo e Gols Contra, com coloração condicional no saldo (Verde/Vermelho).

```dax
VAR GP = 
    CALCULATE(SUM('WorldCupMatches'[Home Team Goals]), USERELATIONSHIP('WorldCupMatches'[Home Team Name], 'Dim_Paises'[Pais])) + 
    CALCULATE(SUM('WorldCupMatches'[Away Team Goals]), USERELATIONSHIP('WorldCupMatches'[Away Team Name], 'Dim_Paises'[Pais]))

VAR GC = 
    CALCULATE(SUM('WorldCupMatches'[Away Team Goals]), USERELATIONSHIP('WorldCupMatches'[Home Team Name], 'Dim_Paises'[Pais])) + 
    CALCULATE(SUM('WorldCupMatches'[Home Team Goals]), USERELATIONSHIP('WorldCupMatches'[Away Team Name], 'Dim_Paises'[Pais]))

VAR Saldo = GP - GC
VAR CorSaldo = IF(Saldo >= 0, "#2ecc71", "#e74c3c") -- Verde ou Vermelho
VAR Sinal = IF(Saldo > 0, "+", "")

RETURN
"<div style='background-color: white; padding: 10px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); border: 1px solid #eee; display: flex; justify-content: space-between; align-items: center; font-family: Segoe UI, sans-serif; height: 100%;'>" &

    -- BLOCO GOLS PRÓ
    "<div style='text-align: center; width: 30%;'>" &
        "<div style='font-size: 11px; color: #666; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 5px;'>GOLS PRÓ</div>" &
        "<div style='font-size: 26px; font-weight: 800; color: #2ecc71;'>⚽ " & GP & "</div>" &
    "</div>" &

    -- DIVISOR NOVO
    "<div style='width: 2px; height: 50px; background-color: #ccc; border-radius: 2px;'></div>" &

    -- BLOCO SALDO (MEIO)
    "<div style='text-align: center; width: 30%;'>" &
        "<div style='font-size: 11px; color: #666; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 5px;'>SALDO</div>" &
        "<div style='font-size: 22px; font-weight: bold; color: " & CorSaldo & ";'>" & Sinal & Saldo & "</div>" &
    "</div>" &

    -- DIVISOR NOVO
    "<div style='width: 2px; height: 50px; background-color: #ccc; border-radius: 2px;'></div>" &

    -- BLOCO GOLS CONTRA
    "<div style='text-align: center; width: 30%;'>" &
        "<div style='font-size: 11px; color: #666; font-weight: 600; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 5px;'>GOLS CONTRA</div>" &
        "<div style='font-size: 26px; font-weight: 800; color: #e74c3c;'>🥅 " & GC & "</div>" &
    "</div>" &

"</div>"
```

### HTML\_Card\_Participacoes

```dax
VAR Quantidade = [Total_Copas_Disputadas]
VAR ListaAnos = [Lista_Anos_Jogados]

RETURN
"<div style='background-color: white; border-radius: 10px; padding: 15px; border: 1px solid #eee; font-family: Segoe UI, sans-serif; box-shadow: 0 2px 5px rgba(0,0,0,0.05);'>" &
    
    -- Cabeçalho com Ícone e Título
    "<div style='display: flex; align-items: center; margin-bottom: 5px;'>" &
        "<div style='font-size: 20px; margin-right: 10px;'>🌍</div>" & -- Ícone Globo
        "<div style='font-size: 14px; color: #888; text-transform: uppercase; font-weight: 600;'>Participações em Copas</div>" &
    "</div>" &

    -- Número Grande
    "<div style='font-size: 40px; font-weight: 900; color: #333; line-height: 1; margin-bottom: 10px;'>" & 
        Quantidade & 
    "</div>" &

    -- Lista de Anos (Texto pequeno com quebra de linha automática)
    "<div style='font-size: 11px; color: #666; line-height: 1.4;'>" & 
        ListaAnos & 
    "</div>" &

"</div>"
```

### Titulo\_Dinamico\_HTML

Header da página que se adapta: mostra "World Cup History" se nada for selecionado, ou o nome e bandeira do país se houver filtro.

```dax
VAR PaisSelecionado = SELECTEDVALUE('Dim_Paises'[Pais_PT])
VAR Bandeira = MAX('Dim_Paises'[URL_Bandeira])

-- 2. Define o HTML base (Estilo da fonte)
VAR EstiloTexto = "font-family: Segoe UI, sans-serif; color: #333;"

RETURN
IF(ISBLANK(PaisSelecionado),

    -- ESTADO 1: NENHUM PAÍS SELECIONADO (Título Genérico)
    "<div style='display: flex; align-items: center; padding-bottom: 10px; border-bottom: 2px solid #eee; " & EstiloTexto & "'>" &
        "<div style='font-size: 32px; margin-right: 10px;'>🏆</div>" & -- Ícone de Troféu
        "<div>" &
            "<div style='font-size: 24px; font-weight: 800;'>World Cup History</div>" &
            "<div style='font-size: 14px; color: #888;'>Selecione um país para ver os detalhes</div>" &
        "</div>" &
    "</div>",

    -- ESTADO 2: PAÍS SELECIONADO (Título Personalizado)
    "<div style='display: flex; align-items: center; padding-bottom: 10px; border-bottom: 2px solid #eee; " & EstiloTexto & "'>" &
        -- Bandeira Grande
        "<img src='" & Bandeira & "' style='width: 60px; height: 60px; border-radius: 50%; border: 3px solid #fff; box-shadow: 0 2px 5px rgba(0,0,0,0.2); margin-right: 15px; object-fit: cover;'>" &
        
        -- Textos
        "<div>" &
            "<div style='font-size: 30px; font-weight: 900; letter-spacing: -1px;'>" & PaisSelecionado & "</div>" &
            "<div style='font-size: 14px; color: #666; font-weight: 500; text-transform: uppercase; letter-spacing: 1px;'>Histórico de Desempenho</div>" &
        "</div>" &
    "</div>"
)
```

### HTML Bandeira

```dax
IF(
    HASONEVALUE('Dim_Paises'[Pais]),
    
    -- SE TIVER UM SELECIONADO: Faz o código normal
    VAR Link = MAX('Dim_Paises'[URL_Bandeira])
    RETURN
    "<div style='display: flex; justify-content: center; align-items: center; height: 100%;'>" &
        "<img src='" & Link & "' style='width: 80px; height: 80px; border-radius: 50%; border: 3px solid #333; object-fit: cover;'>" &
    "</div>",
    
    -- SE NÃO TIVER (ou tiver vários): Retorna vazio (Transparente)
    BLANK()
)
```

### HTML\_Tooltip\_Mapa

Tooltip que aparece ao passar o mouse sobre o mapa, mostrando resumo de títulos e sedes.

```dax
VAR PaisIngles = SELECTEDVALUE('Dim_Paises'[Pais])
VAR PaisPT = SELECTEDVALUE('Dim_Paises'[Pais_PT])
VAR Bandeira = SELECTEDVALUE('Dim_Paises'[URL_Bandeira])

-- Títulos (Anos)
VAR QtdTitulos = 
    CALCULATE(
        COUNTROWS('WorldCups'), 
        'WorldCups'[Winner] = PaisIngles,
        REMOVEFILTERS('WorldCups') 
    )

VAR AnosTitulos = 
    CALCULATE(
        CONCATENATEX(VALUES('WorldCups'[Year]), 'WorldCups'[Year], ", ", 'WorldCups'[Year], ASC), 
        'WorldCups'[Winner] = PaisIngles,
        REMOVEFILTERS('WorldCups')
    )

-- Sedes (Com a correção Korea/Japan)
VAR QtdSede = 
    CALCULATE(
        COUNTROWS('WorldCups'), 
        (
            'WorldCups'[Country] = PaisIngles || 
            ('WorldCups'[Country] = "Korea/Japan" && (PaisIngles = "Japan" || PaisIngles = "South Korea" || PaisIngles = "Korea Republic"))
        ),
        REMOVEFILTERS('WorldCups')
    )

VAR AnosSede = 
    CALCULATE(
        CONCATENATEX(VALUES('WorldCups'[Year]), 'WorldCups'[Year], ", ", 'WorldCups'[Year], ASC), 
        (
            'WorldCups'[Country] = PaisIngles || 
            ('WorldCups'[Country] = "Korea/Japan" && (PaisIngles = "Japan" || PaisIngles = "South Korea" || PaisIngles = "Korea Republic"))
        ),
        REMOVEFILTERS('WorldCups')
    )

-- >>> CORREÇÕES DE PORTUGUÊS (PLURAL/SINGULAR) <<<
VAR TextoTitulos = IF(QtdTitulos = 1, " Título", " Títulos")
VAR TextoVezes = IF(QtdSede = 1, " vez", " vezes")

-- Display Name
VAR NomeDisplay = IF(ISBLANK(PaisPT), PaisIngles, PaisPT)

RETURN
"<div style='background-color: white; padding: 15px; border-radius: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.2); font-family: Segoe UI, sans-serif; width: 250px;'>" &
    
    -- Cabeçalho
    "<div style='display: flex; align-items: center; border-bottom: 1px solid #eee; padding-bottom: 10px; margin-bottom: 10px;'>" &
        "<img src='" & IF(ISBLANK(Bandeira), "[https://via.placeholder.com/30](https://via.placeholder.com/30)", Bandeira) & "' style='width: 40px; height: 30px; border-radius: 4px; border: 1px solid #ddd; object-fit: cover; margin-right: 10px;'>" &
        "<div style='font-size: 18px; font-weight: 800; color: #333;'>" & NomeDisplay & "</div>" &
    "</div>" &

    -- Corpo: Títulos
    IF(QtdTitulos > 0,
        "<div style='margin-bottom: 10px;'>" &
            -- AQUI ENTRA A CORREÇÃO 'TextoTitulos'
            "<div style='font-size: 12px; font-weight: 700; color: #f1c40f; text-transform: uppercase;'>🏆 " & QtdTitulos & TextoTitulos & "</div>" &
            "<div style='font-size: 11px; color: #666;'>" & AnosTitulos & "</div>" &
        "</div>",
        ""
    ) &

    -- Corpo: Sede
    IF(QtdSede > 0,
        "<div>" &
            -- AQUI ENTRA A CORREÇÃO 'TextoVezes'
            "<div style='font-size: 12px; font-weight: 700; color: #34495e; text-transform: uppercase;'>🏠 Sediou " & QtdSede & TextoVezes & "</div>" &
            "<div style='font-size: 11px; color: #666;'>" & AnosSede & "</div>" &
        "</div>",
        ""
    ) &
    
    -- Fallback
    IF(QtdTitulos = 0 && QtdSede = 0, "<div style='color: #999; font-size: 12px; font-style: italic;'>Sem histórico de títulos ou sede.</div>", "") &

"</div>"
```

-----

## 2\. 🧮 Back-End: Regras de Negócio e Estatísticas

Cálculos fundamentais de performance. O principal desafio aqui é que a base de dados separa Mandantes (Home) e Visitantes (Away), exigindo o uso de `USERELATIONSHIP` para unificar as estatísticas de um país independentemente de onde ele jogou.

### Total de Jogos

Calcula o total de partidas somando as aparições como mandante e visitante.

```dax
VAR JogosEmCasa = 
    CALCULATE(
        COUNTROWS('WorldCupMatches'), 
        USERELATIONSHIP('WorldCupMatches'[Home Team Name], 'Dim_Paises'[Pais])
    )

VAR JogosFora = 
    CALCULATE(
        COUNTROWS('WorldCupMatches'), 
        USERELATIONSHIP('WorldCupMatches'[Away Team Name], 'Dim_Paises'[Pais])
    )

RETURN 
    JogosEmCasa + JogosFora
```

### Vitorias

Calcula vitórias verificando se o país selecionado marcou mais gols que o adversário.

```dax
VAR PaisSel = SELECTEDVALUE('Dim_Paises'[Pais])
RETURN
CALCULATE(
    COUNTROWS('WorldCupMatches'),
    FILTER(
        'WorldCupMatches',
        ('WorldCupMatches'[Home Team Name] = PaisSel && 'WorldCupMatches'[Home Team Goals] > 'WorldCupMatches'[Away Team Goals]) ||
        ('WorldCupMatches'[Away Team Name] = PaisSel && 'WorldCupMatches'[Away Team Goals] > 'WorldCupMatches'[Home Team Goals])
    )
)
```

### Empates

```dax
VAR PaisSel = SELECTEDVALUE('Dim_Paises'[Pais])
RETURN
CALCULATE(
    COUNTROWS('WorldCupMatches'),
    FILTER(
        'WorldCupMatches',
        ('WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel) &&
        'WorldCupMatches'[Home Team Goals] = 'WorldCupMatches'[Away Team Goals]
    )
)
```

### Derrotas

```dax
VAR PaisSel = SELECTEDVALUE('Dim_Paises'[Pais])
RETURN
CALCULATE(
    COUNTROWS('WorldCupMatches'),
    FILTER(
        'WorldCupMatches',
        ('WorldCupMatches'[Home Team Name] = PaisSel && 'WorldCupMatches'[Home Team Goals] < 'WorldCupMatches'[Away Team Goals]) ||
        ('WorldCupMatches'[Away Team Name] = PaisSel && 'WorldCupMatches'[Away Team Goals] < 'WorldCupMatches'[Home Team Goals])
    )
)
```

### % Aproveitamento

Fórmula clássica: (Pontos Reais / Pontos Possíveis), considerando 3 pontos por vitória.

```dax
VAR PtsPossiveis = [Total de Jogos] * 3 
VAR PtsConquistados = ([Vitorias] * 3) + ([Empates] * 1)
RETURN
DIVIDE(PtsConquistados, PtsPossiveis, 0)
```

### Total\_Copas\_Disputadas

Conta distintamente os anos (Years) em que o país aparece na tabela de partidas.

```dax
VAR PaisSel = SELECTEDVALUE('Dim_Paises'[Pais])

RETURN
CALCULATE(
    DISTINCTCOUNT('WorldCupMatches'[Year]), -- Conta quantos anos únicos existem
    FILTER(
        'WorldCupMatches',
        'WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel
    )
)
```

### Total\_Gols\_Marcados

```dax
VAR GolsComoMandante = 
    CALCULATE(
        SUM('WorldCupMatches'[Home Team Goals]), 
        USERELATIONSHIP('WorldCupMatches'[Home Team Name], 'Dim_Paises'[Pais])
    )

VAR GolsComoVisitante = 
    CALCULATE(
        SUM('WorldCupMatches'[Away Team Goals]), 
        USERELATIONSHIP('WorldCupMatches'[Away Team Name], 'Dim_Paises'[Pais])
    )

RETURN 
    GolsComoMandante + GolsComoVisitante
```

### Total\_Gols\_Sofridos

Soma os gols feitos pelo *adversário* contra o país selecionado.

```dax
VAR GolsLevadosComoMandante = 
    CALCULATE(
        SUM('WorldCupMatches'[Away Team Goals]), -- Soma gol do adversário
        USERELATIONSHIP('WorldCupMatches'[Home Team Name], 'Dim_Paises'[Pais])
    )

VAR GolsLevadosComoVisitante = 
    CALCULATE(
        SUM('WorldCupMatches'[Home Team Goals]), -- Soma gol do adversário
        USERELATIONSHIP('WorldCupMatches'[Away Team Name], 'Dim_Paises'[Pais])
    )

RETURN 
    GolsLevadosComoMandante + GolsLevadosComoVisitante
```

### Media\_Gols\_Por\_Partida

```dax
DIVIDE(
    SUM('WorldCups'[GoalsScored]), 
    SUM('WorldCups'[MatchesPlayed]), 
    0
)
```

### Total\_Gols\_Jogador

Parses de string para contar gols. A coluna "Event" vem no formato "G40' G50'". A lógica calcula quantos "G" existem removendo-os e comparando o tamanho da string.

```dax
SUMX(
    'WorldCupPlayers',
    VAR Evento = 'WorldCupPlayers'[Event]
    RETURN
    -- O Truque: Tamanho do texto original MENOS tamanho do texto sem a letra "G"
    IF(ISBLANK(Evento), 0, LEN(Evento) - LEN(SUBSTITUTE(Evento, "G", "")))
)
```

### Contagem\_Gols\_Events

Similar à anterior, mas aprimorada para contar também Pênaltis convertidos (P) e ignorar gols contra (OG).

```dax
SUMX(
    'WorldCupPlayers',
    VAR EventoOriginal = 'WorldCupPlayers'[Event]
    
    -- 1. Removemos "OG" (Gol Contra) e "MP" (Pênalti Perdido) para não contar errado
    VAR EventoLimpo = SUBSTITUTE(SUBSTITUTE(EventoOriginal, "OG", ""), "MP", "")
    
    -- 2. Contamos quantos "G" (Gols) sobraram
    VAR QtdGols = LEN(EventoLimpo) - LEN(SUBSTITUTE(EventoLimpo, "G", ""))
    
    -- 3. Contamos quantos "P" (Pênaltis Convertidos) sobraram
    VAR QtdPenaltis = LEN(EventoLimpo) - LEN(SUBSTITUTE(EventoLimpo, "P", ""))
    
    RETURN
    -- Soma tudo. Se for vazio, retorna 0.
    IF(ISBLANK(EventoOriginal), 0, QtdGols + QtdPenaltis)
)
```

-----

## 3\. ⚙️ Lógica de Navegação e Filtros

Medidas de suporte que controlam a interatividade do relatório, garantindo que os filtros de país afetem corretamente o mapa e as listas.

### Filtro\_Jogos\_Do\_Pais

Mágica para filtrar a tabela de partidas no painel lateral. Retorna `1` se o jogo deve aparecer (se o país selecionado é mandante ou visitante) e `0` caso contrário.

```dax
VAR PaisSelecionado = SELECTEDVALUE('Dim_Paises'[Pais])
VAR TimeCasa = SELECTEDVALUE('WorldCupMatches'[Home Team Name])
VAR TimeFora = SELECTEDVALUE('WorldCupMatches'[Away Team Name])

RETURN
-- Se não tiver filtro, mostra tudo (1). Se tiver, só mostra se o país for Casa OU Visitante.
IF(ISBLANK(PaisSelecionado) || TimeCasa = PaisSelecionado || TimeFora = PaisSelecionado, 1, 0)
```

### Lista\_Anos\_Jogados

Gera uma string concatenada (ex: "1958 • 1962 • 1970") para exibição compacta nos cartões.

```dax
VAR PaisSel = SELECTEDVALUE('Dim_Paises'[Pais])

RETURN
CONCATENATEX(
    CALCULATETABLE(
        VALUES('WorldCupMatches'[Year]), -- Pega a lista de anos únicos
        FILTER(
            'WorldCupMatches',
            'WorldCupMatches'[Home Team Name] = PaisSel || 'WorldCupMatches'[Away Team Name] = PaisSel
        )
    ),
    'WorldCupMatches'[Year], -- O texto que será mostrado
    " • ",                   -- O separador (uma bolinha entre os anos)
    'WorldCupMatches'[Year], -- Ordenar por ano
    ASC                      -- Crescente
)
```

### Map\_Color\_Score

Define a lógica de coloração do Mapa (Shape Map).

  * **0:** Padrão (Cinza)
  * **1:** Sede (Azul)
  * **2:** Campeão (Dourado)
    Inclui regra de exceção para a Copa de 2002 (Korea/Japan).

<!-- end list -->

```dax
VAR PaisAtual = SELECTEDVALUE('Dim_Paises'[Pais]) -- Nome em Inglês vindo da Dimensão

-- Verifica se foi Sede NESTE ANO (Com regra especial para 2002)
VAR IsHost = 
    CALCULATE(
        COUNTROWS('WorldCups'), 
        -- Regra 1: O nome bate exatamente (ex: Brazil = Brazil)
        'WorldCups'[Country] = PaisAtual || 
        -- Regra 2 (Exceção 2002): Se a sede for "Korea/Japan", pinta Japão e Coreia
        (
            'WorldCups'[Country] = "Korea/Japan" && 
            (PaisAtual = "Japan" || PaisAtual = "South Korea" || PaisAtual = "Korea Republic")
        )
    ) > 0

-- Verifica se foi Campeão NESTE ANO (Respeita o filtro)
VAR IsWinner = 
    IF(CALCULATE(COUNTROWS('WorldCups'), 'WorldCups'[Winner] = PaisAtual) > 0, 1, 0)

RETURN 
-- Lógica:
-- 1 = Sede (Azul)
-- 2 = Campeão (Dourado - Multipliquei por 2 para dar peso maior)
-- 3 = Sede e Campeão (Dourado Forte)
(IF(IsHost, 1, 0) + IF(IsWinner = 1, 2, 0))
```

```
```
