# Bug Hunting Agent
### Learning-Based Vulnerability Triage and LLM Robustness Under Adversarial Web Content

An applied AI security project investigating how machine learning can improve web vulnerability triage and how adversarial web content can manipulate LLM-based security analysis.

**Project Status:** Research & Development

---

## Overview

Identifying potentially vulnerable endpoints and parameters is an important part of web application security testing. Traditional reconnaissance and vulnerability triage often rely on manually written heuristics, parameter-name patterns, and security tools.

This project explores two connected research problems:

1. **Learning-Based Vulnerability Triage:** Developing a supervised machine learning classifier to identify HTTP inputs that are worth investigating for potential web vulnerabilities, and evaluating its performance against a rule-based baseline.

2. **LLM Robustness Under Adversarial Web Content:** Investigating whether malicious instructions embedded in untrusted web content, including HTML, JavaScript, and HTTP responses, can manipulate an LLM performing vulnerability triage.

The project aims to compare conventional heuristics, trained machine learning models, and LLM-based approaches in terms of detection performance, reliability, and robustness.

## Research Objectives

- Build a labelled dataset of vulnerable and benign HTTP parameters and endpoints.
- Develop structural, lexical, and contextual features for vulnerability classification.
- Implement a rule-based vulnerability triage baseline.
- Train and evaluate supervised machine learning models.
- Investigate prompt injection attacks against LLM-based vulnerability triage.
- Measure the impact of adversarial web content on LLM predictions.
- Compare the performance, robustness, and failure modes of different approaches.

## Research Tracks

### 1. Learning-Based Vulnerability Triage

This track focuses on developing a trainable classifier that predicts whether an HTTP input parameter is worth further security investigation.

Initial vulnerability categories include:

| Category | Description |
|---|---|
| SQL Injection (SQLi) | Potential unsafe interaction with SQL queries |
| Cross-Site Scripting (XSS) | Potential unsafe rendering of user-controlled input |
| Server-Side Request Forgery (SSRF) | Potential manipulation of server-side requests |
| Insecure Direct Object Reference (IDOR) | Potential broken object-level access control |
| Open Redirect | Potential redirection to untrusted destinations |
| Local File Inclusion (LFI) | Potential unsafe local file access or inclusion |

**Planned approach:**

- HTTP parameter extraction and normalisation.
- Dataset construction and ground-truth labelling.
- Feature engineering.
- Rule-based classification.
- Supervised machine learning.
- Performance evaluation and error analysis.

### 2. LLM Robustness Under Adversarial Web Content

This track investigates the reliability of LLMs when analysing untrusted web content.

An attacker may embed malicious instructions inside HTML, JavaScript, or HTTP responses. When an LLM processes this content, it may incorrectly interpret those instructions as trusted directions.

The research will investigate:

- Direct and indirect prompt injection in security-analysis workflows.
- Adversarial instructions embedded in web content.
- Changes in vulnerability-triage predictions under adversarial conditions.
- Attack success rate and model failure patterns.
- Differences between conventional ML classifiers and LLM-based approaches.

The objective is to understand the limitations of LLM-based security analysis and evaluate its reliability under adversarial manipulation.

## Proposed System Architecture

The system is planned around the following components:

```text
          HTTP Input Collection
                   |
                   v
             HTTP Parser
                   |
                   v
          Feature Extraction
                   |
          +--------+--------+
          |                 |
          v                 v
     Rule-Based         ML Classifier
      Baseline                |
          |                   |
          +--------+----------+
                   |
                   v
          Vulnerability Triage
                   |
                   v
          LLM-Based Triage
                   |
                   v
       Adversarial Robustness
             Evaluation
                   |
                   v
          Results & Analysis
```

Reconnaissance tools will serve as supporting infrastructure for collecting HTTP inputs and web content. The primary research contribution lies in model development, evaluation, and robustness analysis.

## Planned Technology Stack

| Area | Technologies |
|---|---|
| Programming Language | Python |
| Machine Learning | Scikit-learn, XGBoost |
| Deep Learning (optional) | Hugging Face Transformers |
| Data Processing | Pandas, NumPy |
| LLM Integration | LLM APIs |
| Evaluation | Scikit-learn metrics |
| Reconnaissance | Subfinder, httpx, Katana, gau |
| Backend | FastAPI |
| Database | SQLite / PostgreSQL |
| Version Control | Git and GitHub |

The final selection of models and technologies will be informed by the literature review and experimental requirements.

## Evaluation Methodology

The project will evaluate the proposed approaches using independent labelled datasets and controlled adversarial test cases.

**Vulnerability triage evaluation:**
- Precision
- Recall
- F1-score
- Confusion matrix
- Precision-recall curves
- Feature importance
- False-positive analysis

The primary objective is to achieve higher precision than the rule-based baseline at matched recall.

**LLM robustness evaluation:**
- Prompt-injection attack success rate
- Prediction changes under adversarial content
- Incorrect triage decisions
- Performance degradation
- Comparative failure analysis

Train, validation, and test data will be separated to reduce data leakage and ensure reliable evaluation.

## Project Roadmap

| Month | Planned Milestone |
|---|---|
| 1 | Literature review and dataset preparation |
| 2 | Feature engineering and rule-based baseline |
| 3 | Machine learning model development |
| 4 | Model evaluation and error analysis |
| 5 | LLM prompt-injection robustness experiments |
| 6 | Comparative evaluation and supporting reconnaissance pipeline |
| 7 | Validation using authorised lab targets |
| 8 | Final evaluation, documentation and project report |

## Ethical Considerations

This project is intended for academic research and authorised security testing.

- Active testing will be restricted to explicitly authorised targets.
- Controlled environments such as OWASP Juice Shop and PortSwigger Web Security Academy will be used for validation.
- No unauthorised exploitation or testing of third-party systems will be conducted.
- The research focuses on vulnerability triage and model robustness rather than autonomous exploitation.

## Current Development Status

This repository will document the research, implementation, experiments, and findings throughout the project.

Planned development stages:

- [ ] Literature review and research gap identification
- [ ] Dataset design and annotation
- [ ] Rule-based baseline implementation
- [ ] Feature extraction pipeline
- [ ] Supervised ML classifier
- [ ] Classifier evaluation
- [ ] LLM-based triage pipeline
- [ ] Adversarial prompt-injection experiments
- [ ] Comparative robustness evaluation
- [ ] Lab validation
- [ ] Final documentation

---

*This project is being developed as a minor project in Applied AI Security, exploring the intersection of web application security, machine learning, and LLM robustness.*
