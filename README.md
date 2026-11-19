# NLP-Regras

Extração de Informações (*Information Extraction*) de contratos em formato Markdown utilizando **Python** e a biblioteca padrão **`re` (Expressões Regulares)**.

---

## 📋 Sobre o Projeto

Este projeto implementa um motor determinístico baseado em Expressões Regulares (Regex) para classificar modelos de contratos extrajudiciais e extrair entidades relevantes (partes envolvidas, documentos, valores financeiros e datas) para uma estrutura padronizada em Dicionário Python / JSON.

### 🎯 Objetivo

Desenvolver um pipeline de extração que:
1. Leia o conteúdo textual de um contrato em formato `.md` (com suporte a codificação UTF-8).
2. Classifique o documento entre **Modelo Simples** e **Modelo Completo** com base na quantidade de cláusulas identificadas.
3. Extraia e normalize os dados cadastrais, financeiros e temporais de forma resiliente a variações de formatação Markdown (`**negrito**`, quebras de linha e variações textuais).

---

## 📄 Templates de Contrato

A classificação estrutural do documento é a primeira etapa do fluxo de automação, pois determina a confiabilidade e o roteamento das regras de extração:

| Modelo de Template | Quantidade de Cláusulas | Comportamento |
| :--- | :---: | :--- |
| **Modelo Simples** | `13` | Contrato padrão de confissão de dívida e parcelamento |
| **Modelo Completo** | `16` | Contrato estendido com cláusulas adicionais |
| **Modelo Desconhecido** | Outro valor | Sinaliza divergência estrutural para revisão |

---

## 🔍 Atributos Extraídos e Regras de Negócio

| Atributo | Descrição | Regra de Negócio / Localização |
| :--- | :--- | :--- |
| **Total de Cláusulas** | Quantidade de seções `## CLÁUSULA ...` | Contagem via divisão/correspondência *case-insensitive* |
| **Modelo de Template** | `Modelo Simples` ou `Modelo Completo` | Definido a partir do total de cláusulas (`13` ou `16`) |
| **Dados do Cliente** | Nome completo, RG (incluindo dígito `X`) e CPF | Extraídos da qualificação do **DEVEDOR** no preâmbulo |
| **Dados do Credor** | Razão Social / Nome da empresa e CNPJ | Extraídos da qualificação do **CREDOR** no preâmbulo |
| **Valor Total** | Valor total líquido e certo da dívida (`float`) | Localizado na **Cláusula Primeira** |
| **Valor da Parcela** | Valor mensal fixo das parcelas (`float`) | Localizado na **Cláusula Quarta** |
| **Data de Assinatura** | Data oficial de assinatura do acordo (`DD/MM/AAAA`) | Localizada na **Cláusula Segunda** (com *fallback* no fecho) |
| **Data de Pagamento / Crédito** | Data prevista para liberação/crédito em conta (`DD/MM/AAAA`) | Localizada na **Cláusula Terceira** |
| **Data de Vencimento da 1ª Parcela** | Data de início do pagamento das parcelas (`DD/MM/AAAA`) | Localizada na **Cláusula Quarta** |

---

## ⚙️ Requisitos Técnicos e Boas Práticas

* **Python 3.8+** (utiliza apenas bibliotecas nativas: `re` e `json`).
* **Leitura Segura de Arquivos:** Abertura explícita com `encoding="utf-8"` para evitar erros de decodificação com acentuação e símbolos (`º`, `§`, `R$`).
* **Resiliência a Variações Textuais:**
  * Uso de `re.IGNORECASE` (`(?i)`) para tolerar variações de caixa (ex.: `## CLÁUSULA PRIMEIRA` e `## cláusula NONA`).
  * Uso de `\s+` entre palavras para suportar quebras de linha arbitrárias no Markdown.
  * Quantificadores opcionais para variações de redação (ex.: `CPF nº` vs. `CPF sob o nº`, `CREDOR, a empresa` vs. `Razão Social`) e marcações de negrito (`**`).
* **Prevenção de *Catastrophic Backtracking*:** Uso de classes de caracteres negadas (como `[^*\r\n,]+?`) e limites explícitos em buscas contextuais.
* **Normalização Numérica:** Conversão de strings monetárias no padrão brasileiro (`"12.450,50"`) para `float` (`12450.5`).

---

## 🚀 Estrutura e Fluxo de Execução

```text
contrato.md
   │
   ▼
[1. Leitura UTF-8]
   │
   ▼
[2. identificar_template()] ──► Conta ocorrências de "## CLÁUSULA" (13 ou 16)
   │
   ▼
[3. Extração por Domínio]
   ├── Cliente: nome, RG, CPF
   ├── Credor: razão social, CNPJ
   ├── Financeiro: valor_total, valor_parcela (convertidos para float)
   └── Datas: assinatura, pagamento/crédito, vencimento 1ª parcela
   │
   ▼
[4. Saída Estruturada (dict / JSON)]
```

### Exemplo de Uso em Python

```python
import json

with open("contrato.md", "r", encoding="utf-8") as arquivo:
    texto_contrato = arquivo.read()

resultado = extrair_dados_contrato(texto_contrato)
print(json.dumps(resultado, ensure_ascii=False, indent=2))
```

---

## 📦 Exemplo de Saída Esperada

```json
{
  "template": "Modelo Simples",
  "clausulas_identificadas": 13,
  "cliente": {
    "nome": "CARLOS EDUARDO DOS SANTOS",
    "rg": "12.345.678-X",
    "cpf": "456.789.123-00"
  },
  "credor": {
    "nome": "SOLUCOES FINANCEIRAS ALPHA LTDA",
    "cnpj": "12.345.678/0001-99"
  },
  "financeiro": {
    "valor_total": 12450.5,
    "valor_parcela": 850.0
  },
  "datas": {
    "data_assinatura": "21/01/2026",
    "data_pagamento": "26/01/2026",
    "data_vencimento_primeira_parcela": "10/03/2026"
  }
}
```
