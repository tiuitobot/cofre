# Cofre — Pipeline de Redação de PII para LLMs Cloud

> **Garantia arquitetural > garantia contratual.**
> O dado sensível é removido *antes* de chegar na cloud. Matematicamente impossível vazar o que nunca saiu.

## O Problema

Instituições brasileiras reguladas (SEFAZ, cartórios, hospitais, tribunais) precisam usar LLMs cloud (Claude, GPT) para analisar documentos com PII (CPF, CNPJ, RG, nomes, endereços). A LGPD proíbe envio desses dados a servidores externos sem consentimento específico.

**Resultado:** ou não usam IA, ou usam com risco jurídico.

## A Solução

Pipeline de 5 camadas que remove PII deterministicamente antes do documento chegar na cloud:

```
Documento original (com PII)
    → Camada 1: Regex determinístico (CPF, CNPJ, etc.) — ~μs
    → Camada 2: Dicionário contextual ("inscrito no CPF sob") — ~μs
    → Camada 3: NER via spaCy pt-BR (nomes, endereços) — ~100ms/pg
    → Camada 4: LLM local revisor residual (só se necessário) — ~2-5s
    → Camada 4.5: Validação cruzada entre camadas
    → Documento redactado (sem PII) → LLM Cloud
    → Resposta com placeholders → Round-trip local → Resultado final
```

**Princípio:** Determinístico primeiro, IA segundo. O LLM local não faz tudo — só verifica o que as camadas determinísticas não cobriram.

## Stack

| Componente | Tecnologia |
|------------|-----------|
| Orquestrador PII | [Microsoft Presidio](https://microsoft.github.io/presidio/) |
| NLP Engine | spaCy pt-BR |
| Custom Recognizers | Regex + checksum + context (CPF, CNPJ, RG, matrícula) |
| Revisor residual | Ministral 14B via Ollama (YAGNI — só se recall <90%) |
| Round-trip | Presidio Anonymizer (de-anonymization nativa) |
| LLM Cloud | Claude / GPT (só vê documento limpo) |

## PII Brasileiros Suportados

| Tipo | Método | Status |
|------|--------|--------|
| CPF | Regex + checksum | 🔲 A implementar |
| CNPJ | Regex + checksum | 🔲 A implementar |
| RG | Dicionário contextual + NER | 🔲 A implementar |
| Matrícula imóvel | Dicionário contextual | 🔲 A implementar |
| Inscrição estadual | Regex + contexto | 🔲 A implementar |
| Título de eleitor | Regex | 🔲 A implementar |
| PIS/PASEP | Regex | 🔲 A implementar |
| CNH | Regex + contexto | 🔲 A implementar |
| Nomes | NER (spaCy) | 🔲 A implementar |
| Endereços | NER + pattern | 🔲 A implementar |

## Segurança

- **Mapa de substituição:** Armazenado LOCAL, criptografado at-rest (GPG/SQLCipher), chmod 600
- **Canary tokens:** IDs falsos plantados pra detectar vazamentos
- **Dry-run mode:** Pipeline completo sem enviar à cloud (validação)
- **Validação cruzada:** Divergência entre camadas = alerta, não auto-redact
- **Isolation:** Cloud nunca vê PII. Round-trip local.

## Quick Start

```bash
# Setup
pip install presidio-analyzer presidio-anonymizer spacy
python -m spacy download pt_core_news_lg

# Testar (a implementar)
python cofre.py --dry-run documento.txt
python cofre.py --redact documento.txt
python cofre.py --roundtrip documento.txt --provider claude
```

## Estrutura

```
cofre/
├── recognizers/          # Custom recognizers BR (CPF, CNPJ, RG, etc.)
├── tests/                # Testes com documentos reais/sintéticos
├── docs/                 # Documentação por recognizer
├── cofre.py              # Pipeline principal
├── config.json           # Configuração (rotas, thresholds)
└── requirements.txt
```

## Arquitetura Completa

O pipeline de mascaramento é a Parte 1. O projeto completo inclui:

- **Roteamento inteligente** (local vs cloud, baseado em triggers)
- **Segurança em profundidade** (7 camadas de redundância)
- **Canary tokens** (detecção proativa de vazamentos)
- **Plano de resposta a incidentes** (5 níveis, LGPD Art. 48)
- **Conformidade LGPD** (pseudonimização, auditabilidade, privacy by design)

Detalhes: ver `plans/cofre-dados-sigilosos/v4-architecture.md` no workspace.

## Contexto

Projeto de Bruno (economista, análise fiscal). Desenvolvido com assistência de IA (Tiuito/OpenClaw).

Aplicação principal: análise de inadimplência ICMS na SEFAZ sem expor dados de contribuintes.

Potencial: qualquer instituição regulada que precisa de IA SOTA com compliance LGPD.

## Licença

MIT

---

*"Use Claude Opus na sua SEFAZ — com compliance LGPD garantido por arquitetura, não por confiança."*
