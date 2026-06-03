歡迎查看我的 [核心程式碼 (main.py)](https://github.com) 檔案。

```python
import random
import math
import time

# ══════════════════════════════════════════════════════════════════
#  共用常數
# ══════════════════════════════════════════════════════════════════
_BASE_TOTAL_RATE = (9.0 / 55.0) * 0.7   # ≈ 11.4545%
_NP = _BASE_TOTAL_RATE                   # 基準機率 (NP)


# ══════════════════════════════════════════════════════════════════
#  狀態矩陣轉蛋系統 (Switch-Matrix Gacha Framework)
# ══════════════════════════════════════════════════════════════════
class GachaSwitchMatrix:
    """
    使用 0 和 1 狀態矩陣的優化轉蛋系統
    0: 使用常態底率 (BASE_TOTAL_RATE)
    1: 使用特殊加成機率 (special_rate)
    概念從第 31 抽開始套用。
    """

    def __init__(self, boost_limit_percentage):
        self.BASE_TOTAL_RATE = _BASE_TOTAL_RATE
        self.MAX_PULLS = 60
        self.TARGET_SCORE = 100

        self.total_pulls = 0
        self.current_score = 0
        self.total_pt_points = 0
        self.is_grand_prize_won = False

        self.grid_status = [True] * 10
        self.hit_grid_count = 0

        # 黑人保護機制相關變數
        self.current_zone_hit_count = 0
        self.streak_zero_zones = 0
        self.current_black_threshold = 2

        # 設定機率加成上限
        if boost_limit_percentage < 0:
            self.MAX_BOOST_LIMIT = 1.0 - self.BASE_TOTAL_RATE
        else:
            self.MAX_BOOST_LIMIT = float(boost_limit_percentage)

        # 賓果連線規則
        self.bingo_rules = [
            ((1, 2, 3), 8), ((4, 5, 6), 20), ((7, 8, 9), 8),
            ((1, 4, 7), 14), ((2, 5, 8), 8), ((3, 6, 9), 20),
            ((1, 5, 9), 14), ((3, 5, 7), 8)
        ]
        self.triggered_lines = [False] * 8

        # 未中獎時的 PT 安慰獎
        self.PT_REWARDS = [1, 3, 5, 10, 15, 20]
        self.PT_CUM_WEIGHTS = [500, 750, 900, 950, 985, 1000]

        # ─── 你的核心新概念實作 ───
        # fission_states 對應第 30~60 抽的狀態（索引 0 對應第 30 抽，索引 30 對應第 60 抽）
        self.fission_states = [0] * 31 
        self.special_rate = self.BASE_TOTAL_RATE
        self._need_rebuild = True

    def _calc_exp_diff(self):
        """計算 Bayesian 期望值差距"""
        n = self.total_pulls
        hit = float(self.hit_grid_count)
        
        if n < 1:
            return 0.0

        # 貝氏平滑公式
        smooth_p = (hit + 20.0 * _NP) / ((n - 1) + 20.0)
        return max(0.0, (n - 1) * (1.36 * _NP - smooth_p))

    def _rebuild_states(self):
        """
        脈衝拉伸版狀態重構 (Pulse-Stretching Switch Matrix)
        結合了舊版的區塊拉伸理論 (bl = remaining / exp_diff) 
        與新版的 0, 1 狀態矩陣架構。
        """
        # 1. 狀態全歸 0
        for i in range(len(self.fission_states)):
            self.fission_states[i] = 0

        if self.total_pulls < 30:
            return

        exp_diff = self._calc_exp_diff()
        # 如果落後極小，直接維持常態底率
        if exp_diff < 1e-12:
            self.special_rate = self.BASE_TOTAL_RATE
            return

        remaining_pulls = max(1, 60 - self.total_pulls)
        
        # 2. 計算單一區塊長度 bl (剩餘空間 / 缺格數)
        bl = remaining_pulls / exp_diff
        
        # 3. 計算特殊抽的機率 (與舊版 dp 的核心邏輯一致，拉高極限爆發力)
        # 基本加成率加上基於 bl 的反彈權重，並限制不超過上限
        max_boost = self.BASE_TOTAL_RATE + self.MAX_BOOST_LIMIT
        prob_boost = self.BASE_TOTAL_RATE + (1.0 / max(1.0, bl))
        self.special_rate = min(max_boost, 2.0 * prob_boost + self.BASE_TOTAL_RATE)

        # 4. ─── 核心裂變：在狀態矩陣中鋪設脈衝斑馬線 ───
        start_matrix_idx = self.total_pulls - 30 + 1
        n_blocks = math.floor(exp_diff)  # 完整區塊數
        k_rem = exp_diff - n_blocks      # 剩餘的小尾巴
        
        current_logical_pos = 0.0

        # A. 鋪設整數區塊
        for _ in range(n_blocks):
            half_bl = bl * 0.5
            # 前一半是 0 (常態底率)，這裡不需要特別寫，因為原本就是 0
            current_logical_pos += half_bl
            
            # 後一半是 1 (特殊加成率)
            for _ in range(int(math.ceil(half_bl))):
                idx = start_matrix_idx + int(math.floor(current_logical_pos))
                if 1 <= idx <= 30:
                    self.fission_states[idx] = 1
                current_logical_pos += 1.0

        # B. 鋪設餘數小尾巴
        if k_rem > 1e-9:
            rem_bl = (k_rem * remaining_pulls) / exp_diff
            half_rem_bl = rem_bl * 0.5
            current_logical_pos += half_rem_bl
            
            for _ in range(int(math.ceil(half_rem_bl))):
                idx = start_matrix_idx + int(math.floor(current_logical_pos))
                if 1 <= idx <= 30:
                    self.fission_states[idx] = 1
                current_logical_pos += 1.0

        self._need_rebuild = False



    def check_bingo_and_update_score(self):
        added_score = 0
        for idx, (line, score_reward) in enumerate(self.bingo_rules):
            if self.triggered_lines[idx]:
                continue
            if not self.grid_status[line[0]] and not self.grid_status[line[1]] and not self.grid_status[line[2]]:
                self.triggered_lines[idx] = True
                added_score += score_reward
        self.current_score += added_score

    def trigger_extra_black_compensation(self):
        active_items = [i for i in range(1, 10) if self.grid_status[i]]
        if not active_items:
            return
        chosen_item = random.choice(active_items)
        self.grid_status[chosen_item] = False
        self.hit_grid_count += 1
        self.current_zone_hit_count += 1
        self.check_bingo_and_update_score()
        # 中獎了，通知系統需要重構狀態
        self._need_rebuild = True

    def simulate_one_pull(self):
        # 第 60 抽強制保底大獎
        if self.total_pulls == 59:
            self.total_pulls += 1
            self.is_grand_prize_won = True
            if self.current_score < self.TARGET_SCORE:
                self.current_score = self.TARGET_SCORE
            self.total_pt_points += 20
            return

        # 只要狀態需要更新，就呼叫重構函式
        if self.total_pulls >= 30 and self._need_rebuild:
            self._rebuild_states()

        self.total_pulls += 1

        # ─── 根據 0 和 1 狀態判定當前中獎率 ───
        if self.total_pulls <= 30:
            current_total_rate = self.BASE_TOTAL_RATE
        else:
            state_idx = self.total_pulls - 30
            # 讀取狀態矩陣：0 使用底率，1 使用特殊加成率
            if self.fission_states[state_idx] == 1:
                current_total_rate = self.special_rate
            else:
                current_total_rate = self.BASE_TOTAL_RATE

        # 開始抽卡判定
        rand_val = random.random()
        if rand_val < current_total_rate:
            active_items = [i for i in range(1, 10) if self.grid_status[i]]
            if active_items:
                hit_item = random.choice(active_items)
                self.grid_status[hit_item] = False
                self.hit_grid_count += 1
                self.current_zone_hit_count += 1
                self.check_bingo_and_update_score()
                # 實際中獎了，下一抽前必須重構狀態
                self._need_rebuild = True
        else:
            pt_gained = random.choices(self.PT_REWARDS, cum_weights=self.PT_CUM_WEIGHTS, k=1)[0]
            self.total_pt_points += pt_gained

        # 每 5 抽檢查連續未中獎補償 (黑人保護)
        if self.total_pulls % 5 == 0:
            if self.current_zone_hit_count > 0:
                self.streak_zero_zones = 0
            else:
                self.streak_zero_zones += 1

            if self.streak_zero_zones >= self.current_black_threshold:
                self.trigger_extra_black_compensation()
                self.current_black_threshold += 1
                self.streak_zero_zones = 0

            self.current_zone_hit_count = 0

        if self.current_score >= self.TARGET_SCORE:
            self.is_grand_prize_won = True


# ══════════════════════════════════════════════════════════════════
#  蒙地卡羅壓測流程
# ══════════════════════════════════════════════════════════════════
def run_single_version(boost, sims):
    hard_pity_count = 0
    total_pulls_sum = 0
    total_pt_sum    = 0

    for _ in range(sims):
        pool = GachaSwitchMatrix(boost_limit_percentage=boost)
        is_pity = False
        for _ in range(60):
            if pool.total_pulls == 59 and pool.current_score < pool.TARGET_SCORE:
                is_pity = True
            pool.simulate_one_pull()
            if pool.is_grand_prize_won:
                break
        if is_pity:
            hard_pity_count += 1
        total_pulls_sum += pool.total_pulls
        total_pt_sum    += pool.total_pt_points

    avg_pulls  = total_pulls_sum / sims
    avg_pt     = total_pt_sum    / sims
    pity_rate  = hard_pity_count / sims * 100.0
    return avg_pulls, avg_pt, pity_rate


def run_simulation_report(sims_per_loop=100_000):
    mock        = GachaSwitchMatrix(-1.0)
    real_limit  = 1.0 - mock.BASE_TOTAL_RATE
    test_cases  = [0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, -1.0]

    W = 80
    print("=" * W)
    print(f"🚀  全新狀態矩陣 (Switch-Matrix) 轉蛋演算法壓測  ｜  每組樣本: {sims_per_loop:,} 次")
    print(f"    底率 NP ≈ {mock.BASE_TOTAL_RATE*100:.4f}%  ｜  最大補償上限 ≈ {real_limit*100:.4f}%")
    print("=" * W)

    print(f"  {'加成上限設定':<28}{'平均抽數':>12}{'平均 PT':>14}{'硬保底率':>16}")
    print("-" * W)

    start = time.time()
    for case in test_cases:
        label = (f"固定上限 +{case*100:.1f}%" if case >= 0
                 else f"最大上限 +{real_limit*100:.2f}%")

        b_pulls, b_pt, b_pity = run_single_version(case, sims_per_loop)
        print(f"  {label:<28}{b_pulls:>12.2f}{b_pt:>14.2f}{b_pity:>15.4f}%")

    print("=" * W)
    print(f"⏱️  壓測完畢！總耗時: {time.time()-start:.2f} 秒")


if __name__ == "__main__":
    run_simulation_report(sims_per_loop=100_00)

