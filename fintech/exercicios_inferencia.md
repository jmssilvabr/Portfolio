# EXERCÍCIOS DE INFERÊNCIA ESTATÍSTICA — BASE FINTECH SINTÉTICA
## Como usar: carregue os 3 CSVs e resolva os exercícios abaixo com Python/R

```python
import pandas as pd, numpy as np
from scipy import stats

macro     = pd.read_csv("macro.csv")
clientes  = pd.read_csv("clientes.csv")
contratos = pd.read_csv("contratos.csv")
df = contratos.merge(clientes, on="id_cliente")
```

---

## BLOCO 1 — Estimadores e Propriedades

### Ex 1.1 — Vício e Consistência
Estime a renda média populacional usando amostras de tamanho n=30, 100, 300 e 1000.
Compare o estimador de média amostral com a mediana como estimadores da renda.
→ Qual é viciado? Qual é mais eficiente para uma distribuição log-normal?

```python
rendas = clientes["renda_mensal"].values
for n in [30, 100, 300, 1000]:
    amostras = [np.mean(np.random.choice(rendas, n)) for _ in range(2000)]
    print(f"n={n:4d} | E[x̄]={np.mean(amostras):.0f} | Var={np.var(amostras):.0f}")
```

### Ex 1.2 — Eficiência Relativa
Compare a variância de dois estimadores da taxa de juros:
- Estimador A: média aritmética de taxa_juros_am
- Estimador B: mediana de taxa_juros_am
Calcule a eficiência relativa = Var(B)/Var(A) com bootstrap (B=2000 amostras, n=200).

### Ex 1.3 — Consistência do Score
Mostre empiricamente que a média amostral do score_bureau_entrada é consistente:
plote o viés estimado e o EQM para n = 10, 50, 100, 500, 1000, 5000.

---

## BLOCO 2 — Método dos Momentos (MoM)

### Ex 2.1 — Ajuste Log-Normal para Renda
A renda segue Log-Normal(μ, σ²). Estime μ e σ pelo MoM:
- E[X] = exp(μ + σ²/2)  →  log(E[X]) - σ²/2 = μ
- Var[X] = [exp(σ²) - 1] · exp(2μ + σ²)

```python
m1 = clientes["renda_mensal"].mean()
m2 = clientes["renda_mensal"].var()
# resolva o sistema para mu e sigma
sigma2_hat = np.log(m2 / m1**2 + 1)
mu_hat     = np.log(m1) - sigma2_hat / 2
print(f"MoM: mu={mu_hat:.4f}, sigma={np.sqrt(sigma2_hat):.4f}")
```

Compare com o estimador de MLE (próximo bloco). Qual é mais eficiente?

### Ex 2.2 — Ajuste Poisson para número de contratos por cliente
Cada cliente pode ter múltiplos contratos. Conte contratos por cliente e ajuste
uma Poisson pelo MoM. Parâmetro λ = E[X] = Var[X] para Poisson pura — verifique
se a equidispersão é válida nos dados.

---

## BLOCO 3 — Máxima Verossimilhança (MLE)

### Ex 3.1 — MLE da Taxa de Juros (Normal)
Assuma taxa_juros_am ~ N(μ, σ²). Derive e calcule analiticamente:
- μ̂_MLE = x̄
- σ̂²_MLE = (1/n) · Σ(xi - x̄)²  [viciado — por que?]
- σ̂²_não_viciado = s² = (1/(n-1)) · Σ(xi - x̄)²

```python
taxas = contratos["taxa_juros_am"].values
mu_mle    = taxas.mean()
sigma_mle = taxas.std(ddof=0)   # MLE (viciado)
sigma_unb = taxas.std(ddof=1)   # não viciado
```

### Ex 3.2 — MLE Log-Normal para Valor Contratado
Minimize a log-verossimilhança negativa numericamente com scipy.optimize:

```python
from scipy.optimize import minimize
from scipy.stats import lognorm

vals = np.log(contratos["valor_contratado"].values)

def neg_loglik(params):
    mu, sigma = params
    return -np.sum(stats.norm.logpdf(vals, loc=mu, scale=sigma))

res = minimize(neg_loglik, x0=[8.0, 0.8], method="Nelder-Mead")
mu_hat, sigma_hat = res.x
print(f"MLE: mu={mu_hat:.4f}, sigma={sigma_hat:.4f}")
# Compare com estimativa fechada:
print(f"Analítico: mu={vals.mean():.4f}, sigma={vals.std():.4f}")
```

### Ex 3.3 — MLE Exponencial para Dias de Atraso (inadimplentes)
Filtre apenas contratos com dias_atraso > 0. Assuma Exponencial(λ).
MLE: λ̂ = 1/x̄. Calcule o intervalo de confiança para λ usando Fisher Information.

### Ex 3.4 — Comparação MoM vs MLE
Para a renda_mensal log-normal, compare os estimadores MoM e MLE de μ e σ em termos
de viés e variância via simulação (amostras repetidas de n=100).

---

## BLOCO 4 — Intervalos de Confiança

### Ex 4.1 — IC para Renda Média (σ desconhecida → t de Student)
Construa IC de 90%, 95% e 99% para a renda média dos clientes.
Interprete cada intervalo em linguagem de negócio.

```python
n = len(clientes)
xbar = clientes["renda_mensal"].mean()
s    = clientes["renda_mensal"].std(ddof=1)
for alpha in [0.10, 0.05, 0.01]:
    t_crit = stats.t.ppf(1 - alpha/2, df=n-1)
    margem = t_crit * s / np.sqrt(n)
    print(f"IC {int((1-alpha)*100)}%: [{xbar-margem:.0f}, {xbar+margem:.0f}]")
```

### Ex 4.2 — IC para Proporção de Inadimplentes (IC de Wilson)
Estime a proporção de contratos com dias_atraso > 30 com IC de 95% pelo método de Wilson.
Compare com o método de Wald. Para qual n o método de Wald fica confiável?

### Ex 4.3 — IC para Diferença de Médias (Segmentos)
Compare a renda_mensal média entre clientes "Standard" e "Premium".
Use teste de Welch (variâncias diferentes). Construa o IC para μ_Premium - μ_Standard.

### Ex 4.4 — IC Bootstrap para a Mediana do Score
Use bootstrap (B=5000) para construir um IC de 95% para a mediana do score_bureau_entrada.
Compare com o IC paramétrico baseado em ordem.

```python
scores = clientes["score_bureau_entrada"].values
boot_medians = [np.median(np.random.choice(scores, len(scores), replace=True))
                for _ in range(5000)]
ic_low, ic_high = np.percentile(boot_medians, [2.5, 97.5])
print(f"IC Bootstrap 95% para mediana: [{ic_low:.1f}, {ic_high:.1f}]")
```

### Ex 4.5 — IC para a Taxa de Juros Média por Produto
Para cada tipo_produto, construa IC de 95% para taxa_juros_am média.
Plote os intervalos (forest plot) e interprete as diferenças de negócio.

---

## BLOCO 5 — Testes de Hipóteses

### Ex 5.1 — Teste Z: O Score Médio dos Clientes Está Acima de 550?
H₀: μ_score = 550 vs H₁: μ_score > 550 (unilateral)
Use a distribuição Normal (n grande). Calcule o p-value e tome decisão com α=0.05.

```python
score = clientes["score_bureau_entrada"].values
z = (score.mean() - 550) / (score.std() / np.sqrt(len(score)))
p_value = 1 - stats.norm.cdf(z)
print(f"Z = {z:.3f}, p-value = {p_value:.4f}")
```

### Ex 5.2 — Teste t: Renda Difere entre Homens e Mulheres?
H₀: μ_M = μ_F vs H₁: μ_M ≠ μ_F (bilateral)
Use o teste de Welch. Calcule potência do teste (power) para detectar diferença de R$300.

### Ex 5.3 — Erro Tipo I e II na Detecção de Inadimplência
Simule um "classificador simples": alerta de inadimplência se score < 500.
Calcule a matriz de confusão, taxa de Falso Positivo (Erro Tipo I) e
Falso Negativo (Erro Tipo II). Varie o threshold e plote a curva ROC manualmente.

### Ex 5.4 — Teste Qui-Quadrado: Independência de Canal × Inadimplência
Existe associação entre o canal_aquisicao e ter dias_atraso > 30?

```python
df["inadimplente"] = (df["dias_atraso"] > 30).astype(int)
tabela = pd.crosstab(df["canal_aquisicao"], df["inadimplente"])
chi2, p, dof, expected = stats.chi2_contingency(tabela)
print(f"χ² = {chi2:.3f}, p-value = {p:.4f}, graus de liberdade = {dof}")
```

### Ex 5.5 — ANOVA: Score Médio Difere Entre Segmentos?
H₀: μ_Standard = μ_Plus = μ_Premium = μ_Private
Use ANOVA one-way + Tukey HSD para comparações múltiplas.

### Ex 5.6 — Teste de Poder
Para detectar uma redução de 0,5pp na taxa de inadimplência (de 8% para 7,5%),
qual o tamanho mínimo de amostra necessário com α=0.05 e poder=0.80?

```python
from statsmodels.stats.power import NormalIndPower
effect_size = (0.08 - 0.075) / np.sqrt(0.08 * 0.92)  # Cohen's h aproximado
analise = NormalIndPower()
n_min = analise.solve_power(effect_size=effect_size, alpha=0.05, power=0.80)
print(f"n mínimo por grupo: {n_min:.0f}")
```

### Ex 5.7 — p-hacking e Múltiplos Testes
Faça 20 testes t comparando taxa_juros_am entre estados aleatórios.
Quantos serão significativos com α=0.05 apenas por chance?
Aplique correção de Bonferroni e Benjamini-Hochberg. Qual a diferença?

---

## BLOCO 6 — Inferência Bayesiana

### Ex 6.1 — Estimação Bayesiana da Taxa de Inadimplência
Defina Prior Beta(α=2, β=20) para π (taxa de inadimplência).
Após observar k inadimplentes em n contratos, compute a Posteriori Beta(α+k, β+n-k).

```python
# Prior: Beta(2, 20) → crença inicial: inadimplência em torno de 2/22 ≈ 9%
alpha_prior, beta_prior = 2, 20

inadimplentes = (contratos["dias_atraso"] > 90).sum()
n_total       = len(contratos)

alpha_post = alpha_prior + inadimplentes
beta_post  = beta_prior  + n_total - inadimplentes

media_post = alpha_post / (alpha_post + beta_post)
ic_95 = stats.beta.ppf([0.025, 0.975], alpha_post, beta_post)
print(f"Posteriori: Beta({alpha_post}, {beta_post})")
print(f"Média = {media_post:.4f}, IC 95% = [{ic_95[0]:.4f}, {ic_95[1]:.4f}]")
```

Compare com o estimador de MLE (frequentista). Como o prior afeta o resultado
quando n é pequeno vs grande?

### Ex 6.2 — Prior Informativo vs Não-Informativo
Use Prior Beta(1,1) (uniforme) vs Beta(2,20) (informativo).
Com n=50 observações, qual a diferença nas posterioris? Com n=5000?
→ Ilustre o conceito de que o dado "domina" o prior com amostras grandes.

### Ex 6.3 — Estimação Bayesiana da Renda Média (Normal-Normal)
Assumindo renda_mensal ~ N(μ, σ²=3500²) com σ² conhecida:
Prior: μ ~ N(μ₀=4000, τ²=1000²)
Calcule a Posteriori e compare com o IC frequentista.

---

## BLOCO 7 — Exercício Integrador

### Ex 7.1 — Pipeline Completo de Inferência

**Contexto de negócio:** A diretoria quer saber se a taxa de inadimplência (dias_atraso > 90)
dos clientes captados pelo canal "App" é diferente dos captados por "Parceiro".

**Tarefas:**
1. Defina claramente H₀ e H₁
2. Escolha o teste adequado (justifique)
3. Verifique as suposições do teste
4. Calcule o tamanho amostral necessário para poder = 0.80
5. Execute o teste e reporte o p-value
6. Construa IC de 95% para a diferença de proporções
7. Conclua em linguagem de negócio (não estatística)
8. Refaça a análise com abordagem bayesiana (Prior Beta(1,1))
9. Compare as conclusões frequentista vs bayesiana

