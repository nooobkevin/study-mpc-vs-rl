# CONTEXT — 多分布輸入 × 異分布噪聲下的非線性 ODE 控制比較

本倉庫**只**記錄與實作本實驗。禁止另開 `docs/` 設計文檔；計劃以本檔為準。

---

## Q（實驗問題）

是否可以設計一個非線性 ODE 系統，輸入是從不同分布產生數據（Gaussian、Student-t、Lorentzian 等）加不同噪聲（噪聲也有不同分布，不一定 Gaussian，不一定和數據分布一樣），然後對比不同控制方法能否在不同數據–噪聲分布下達成控制目標（最大化某收益／最小化某 cost）？可以有一個極端例子：輸入和噪聲分布是完全隨機（白噪聲？）？

---

## A 總判斷

可以做，且適合當本 project 核心。須先拆開「從不同分布產生數據」與「完全隨機（白噪聲）」混在一起的概念。

---

## 一、三個維度

| 維度 | 意思 | 例子 |
|------|------|------|
| 邊際分布 | 單一時刻的值／尾部 | Gaussian、Student-t、Cauchy（Lorentzian）、Laplace、均勻、雙峰混合 |
| 時間結構 | 時刻間相關／可否預測 | 白噪聲、OU/AR(1) 有色、regime switching、偶發衝擊 |
| 進入位置 | 噪聲加在哪 | 外生輸入 \(d\)、過程噪聲 \(w\)、量測噪聲 \(v\) |

要點：

1. **白噪聲是時間結構，與邊際分布無關。**「完全隨機」拆成：不可預測（白）、極重尾（Cauchy）、連分布族每局隨機（見第六節 E1–E3）。
2. **系統是濾波器。**有限變異數白輸入經耗散動力學後狀態常被「高斯化」。差易顯在：無限變異數／穩定分布（Cauchy、α-stable）；量測噪聲；閾值／失效事件。

---

## 二、受控系統（建議）：飽和致動器 + 雙穩態振子

\[
\begin{aligned}
\dot x_1 &= x_2 \\
\dot x_2 &= -\gamma x_2 + x_1 - x_1^3 + \operatorname{sat}(u) + d(t) \\
y_k &= h(x_1(t_k)) + v_k,\quad h(x)=\tanh(\kappa x)\ \text{或}\ x
\end{aligned}
\]

- 目標：維持右側井 \(x_1\approx +1\)；\(\ell=(x_1-1)^2+\lambda u^2\)；失效 \(x_1<0\)。
- 可改經濟收益：\(r=p\cdot x_1-c\cdot|u|\)，失效大額罰款。
- 旋鈕：\(\gamma\)、勢壘 \(a x-x^3\)、\(u_{\max}\)、\(\kappa\)。
- 模型失配（僅 MB）：\(\gamma\) 偏差，或以 \(\sin\) 恢復力代替三次項。

---

## 三、擾動／噪聲生成

1. **尺度**：Cauchy 無變異數 → 用 IQR 或 MAD 對齊；可另報 99% 分位對齊。
2. **時間**：白＝每步獨立；有色 AR(1)：\(d_k=\phi d_{k-1}+\sqrt{1-\phi^2}\varepsilon_k\)（非穩定分布濾波後邊際會變）。
3. **離散**：建議 ZOH + RK4，寫明 \(\Delta t\)；連續 SDE 時 Gaussian 用 EM（\(\sqrt{\Delta t}\)），α-stable 增量 \(\Delta t^{1/\alpha}\)。
4. **族**：gauss、laplace、t3、t1.5、cauchy、uniform；另雙峰混合 \(0.5\mathcal N(-m,s)+0.5\mathcal N(m,s)\)。

---

## 四、重尾方法論

| 問題 | 處理 |
|------|------|
| 二次成本期望可能不存在 | 中位數、分位數、失效機率；CVaR 只對有界／截斷成本 |
| KF 離群崩潰 | MB 同時放 naive KF 與 robust filter |
| RL reward／TD 重尾 | clipping 或 Huber，報告中說明 |
| 訓練≠測試 | 訓練×測試交叉矩陣 |

關鍵對照：Cauchy 量測下 naive KF vs robust filter vs 帶歷史窗 RL。

---

## 五、實驗矩陣

- **層 1**：單因子（其餘 Gaussian 白）— 擾動分布×強度；量測分布×強度；時間結構（白、OU φ=0.5/0.9/0.99、regime switching）。
- **層 2**：擾動分布 × 量測噪聲分布熱圖（勝出＋差距＋CI）。
- **層 3**：訓練×測試交叉；關鍵格 Gaussian 訓練 → Cauchy 測試。

方法：PID、LQR/MPC+KF、NMPC+EKF/UKF／robust、SAC 變體；補 **robust-filter MPC**、**SAC＋分布隨機化**。Oracle 下界；低 SNR 開環 sanity。

---

## 六、極端與 sanity

| ID | 設定 | 檢驗 |
|----|------|------|
| E1 | 白擾動（任意邊際） | 預測／前饋價值 |
| E2 | Cauchy／α-stable（α&lt;1.5） | 離群／跳躍韌性 |
| E3 | 每 episode 抽分布族＋參數 | 未知噪聲模型韌性 |

- SNR→0：全員開環。
- 擾動與量測同分布白：頻譜可分 → 模型先驗價值。
- 線性＋Gaussian 白＋無約束：LQG 錨點（MPC+KF≈Oracle）。

---

## 七、假設

1. **H1**：有限變異數輸入端差異小（狀態高斯化）；差主要在量測端與失效機率。
2. **H2**：Cauchy 量測：naive MPC+KF &lt; 歷史窗 RL &lt; MPC＋robust filter。
3. **H3**：白擾動下 MPC 優勢主要來自自身動態預測與約束；有色且估擾動進模型時優勢擴大。
4. **H4**：Gaussian 訓練 RL 在 Cauchy 上失效升幅大於 offset-free MPC；分布隨機化縮差距但標稱變差。

產出：附機制解釋的相圖，非純排名。

---

## 倉庫規則

- 只實作本實驗；不保留無關舊 MVP／handoff。
- **禁止寫 `docs/`**；計劃只在本 `CONTEXT.md`。
- 公開倉庫：勿寫入真實廠名／FO–RO／污水品牌。
