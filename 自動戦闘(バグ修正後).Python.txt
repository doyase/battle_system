import random
import copy

#戦闘ログ
battle_log = []

#キャラクターの行動サイクル
ACTION_CYCLE = {
    1:"skill1",
    2:"skill2",
    3:"attack",
    4:"attack"
}

#戦闘ログを記録
def log_event(event):    
    battle_log.append(event)

# スキル発動判定（ターンごとのスキル使用可否を判定）
def can_use_skill(turn):    
    return ACTION_CYCLE.get(turn % 4, "attack")in["skill1", "skill2"]

# 戦闘ターンの処理
def battle_turn(turn, players, enemies):
    """1ターンの戦闘処理の流れ:
       1. ワタシラガの生存チェック（戦闘不能時にまどか参戦）
       2. 味方の行動（攻撃 or スキル）
       3. 敵の行動（攻撃 or スキル）
       4. ターン終了処理（バフ更新など）"""
    
    global wata_fallen    
    wata_fallen = False

    """1ターンの戦闘処理"""
    log_event(f"==={turn}ターン目===")

    # 🔹 ワタシラガの生存状況を確認（毎ターンチェックする）
    wata_is_dead = any(p.name == "ワタシラガ" and not p.is_alive() for p in players)
    already_added = any(p.name == "アルティメットまどか" for p in players)

    # 🔥 「ワタシラガが倒れた」かつ「アルティメットまどかが未参戦」なら仲間追加
    if wata_is_dead and not already_added and not wata_fallen:
        log_event("薄れゆく意識の中、ワタシラガはほむらの勝利を祈った・・・")
        log_event("するとワタシラガの身体を借りてなんとまどかが駆け付けた！？")
        new_ally = Player("アルティメットまどか", 200000, 1500, 750, SKILLS["tenzyouno_inori"], SKILLS["connect"])
        players.append(new_ally)
        wata_fallen = True        
        log_event("✨ アルティメットまどかが降臨し、奇跡を起こす・・・")
        use_skill(new_ally, SKILLS["tenzyouno_inori"], players, players)
        
    # 敵の生存チェック（全滅していたら戦闘終了）
    alive_enemies = [e for e in enemies if e.is_alive()]
    if not alive_enemies:
            log_event("🏆 敵を全滅させた！")
            return

    #味方の行動
    for player in players[:]:
        if not player.is_alive():
            continue
        
        #味方の攻撃またはスキル
        action = ACTION_CYCLE.get(turn % 4, "attack")

        if action == "skill1" and turn % 4 == 1:  # 💡 1ターンに1回だけ発動
            use_skill(player, player.skill1, alive_enemies, players)
        elif action == "skill2" and turn % 4 == 2:
            use_skill(player, player.skill2, alive_enemies, players)
        else:
            player.attack_enemy(random.choice(alive_enemies))
            # 🔽 ここでbreakを入れると味方が1人しか行動しなくなるのでコメントアウト
        #break

        # **💡 味方が攻撃した後、再度全滅チェック**
        if not any(e.is_alive() for e in enemies):
            log_event("🏆 敵を全滅させた！")
            return
    
    #敵の行動
    for enemy in enemies:
        if not enemy.is_alive():
            continue

        #ワルプルギスの夜がHP半分以下でバフ発動
        if enemy.name =="ワルプルギスの夜" and enemy.hp <= enemy.maxhp //2:
            if "rage" not in enemy.buffs:
                log_event(f"🔥 {enemy.name} は反転！遂に本気を出した・・・。 攻撃力 & 防御力 UP！")
                use_skill(enemy, SKILLS["rage_buff"], [enemy], enemies)
                enemy.buffs["rage"] = {"amount":10.0, "turns_left": 999}
        
        # **💡 プレイヤー全滅なら即終了**
        alive_players = [p for p in players if p.is_alive()]
        if not alive_players:
            log_event("💀 プレイヤーが全滅した！")
            return  

        # 敵の攻撃またはスキル
        action = ACTION_CYCLE.get(turn % 4, "attack")
        if action == "skill1" and turn % 4 == 1:
            use_skill(enemy, enemy.skill1, alive_players, enemies)
        elif action == "skill2" and turn % 4 == 2:
            use_skill(enemy, enemy.skill2, alive_players, enemies)
        else:
            enemy.attack_player(random.choice(alive_players))
    
    
    #ダメージ処理&ターン終了処理
    end_turn(players, enemies)


class Character:
    def __init__(self, name, hp, attack, defence, skill1, skill2):
        self.name = name
        self.maxhp = hp
        self.hp = hp        
        self.attack = attack
        self.defence = defence
        self.skill1 = skill1
        self.skill2 = skill2
        self.buffs = {}

    def take_damage(self, damage):
        """ダメージ処理"""
        defence_multiplier = self.buffs.get("defence", {}).get("amount", 1.0)# バフがなければ1.0
        effective_defence = int(self.defence * defence_multiplier)# 防御力をバフ倍率で強化

        actual_damage = max(1, damage - self.defence)
        self.hp = max(0, self.hp - actual_damage)
        log_event(f"{self.name}は{actual_damage}のダメージを受けた！(HP:{self.hp})")
        if self.hp == 0:
            log_event(f"{self.name}は倒れた！")

    def is_alive(self):
            return self.hp > 0
        
    def attack_enemy(self, enemy):
        """通常攻撃"""
        damage = random.randint(self.attack - 2, self.attack + 2)
        log_event(f"{self.name}の攻撃！")
        enemy.take_damage(damage)
   

    def attack_player(self, player):
        """敵がプレイヤーを攻撃する"""
        damage = random.randint(self.attack - 2, self.attack + 2)
        log_event(f"{self.name} の攻撃！")
        player.take_damage(damage)

class Enemy(Character):
    pass

class Player(Character):
    pass


def deal_damage(base_damage, multiplier=1.0):
    
    return int(base_damage * multiplier * (random.uniform(0.8, 1.2)))  # 乱数で揺らぎをつける

def use_skill(user, skill, target_group, user_group):

    """スキル発動処理"""
    if skill is None:
        return
    
    log_event(f"✨ {user.name} の {skill['name']}")

    if skill["type"] == "attack":
        base_damage = skill["base_damage"]
        for target in target_group:
            if target.is_alive():
                for hit in skill['hits']:
                    if target.is_alive():
                        if isinstance(hit, (int, float)):  # 数値ならダメージ計算
                            damage = user.attack * deal_damage(base_damage, float(hit))
                            target.take_damage(damage)                    


    #🔹 もしスキルに "next_skill" が設定されていれば、次のスキルを発動
    if "next_skill" in skill:
        next_skill = SKILLS[skill["next_skill"]]
        log_event(f"🔥 連携技発動！ {next_skill['name']}！")
        use_skill(user, next_skill, target_group, user_group)                                    
                    
    elif skill ["type"] == "heal":
        for user in user_group:
                if user.is_alive():
                    user.hp = min(user.maxhp, user.hp + skill["heal_amount"])
                    log_event(f"💖 {user.name} は {skill['heal_amount']} 回復！（HP: {user.hp}/{user.maxhp}）")

    elif skill["type"] == "buff":
        for user in user_group:
            if user.is_alive():
                if isinstance(skill["buff_amount"], dict):
                    for buff_type, amount in skill["buff_amount"].items():
                        user.buffs[buff_type] = {"amount": amount,
                        "turns_left": skill["duration"]}
                        log_event(f"🛡️ {user.name} は {buff_type} が {skill['buff_amount']} 倍になった！（{skill['duration']}ターン）")
                else:
                    user.buffs[skill["buff_type"]] = {
                    "amount": skill["buff_amount"],
                    "turns_left": skill["duration"]
                    }
                    log_event(f"🛡️ {user.name} の {skill['buff_type']} が {skill['buff_amount']} 倍になった！（{skill['duration']}ターン）")
  
def update_buffs(character):
    """ターンごとにバフの効果を減らす"""
    for buff in list(character.buffs.keys()):
        
        if "turns_left" not in character.buffs[buff]:  # ✅ もし `turns_left` がなかったらエラーを出す
            print(f"⚠️ エラー: {character.name} の {buff} に 'turns_left' がありません！")
            continue  # エラー回避のためスキップ
        character.buffs[buff]["turns_left"] -= 1
        if character.buffs[buff]["turns_left"] <= 0:
            del character.buffs[buff]#バフが切れたら削除
            log_event(f"⏳{character.name}の{buff}の効果が切れた！")


def end_turn(players, enemies):
    """ターン終了時の処理（MP回復、バフ解除、状態異常処理など）"""
    
   # 🔥 MP回復（生存しているキャラのみ）
    for char in players + enemies:
        # バフの効果終了処理
        update_buffs(char)
    
     

    # 🔥 戦闘ログの整理（ターンごとにログをクリア）
    for log in battle_log:
        print(log)
    battle_log.clear()  # 1ターン分のログを表示した後、リセット

    # 空になったキャラのバフデータを削除
    for char in players + enemies:
        char.buffs = {k: v for k, v in char.buffs.items() if v}

SKILLS = {
    "headshot": {"name": "ヘッドショット", "type": "attack", "base_damage": 50, "hits":[1.0]},
    "shooting_star": {"name": "シューティングスター", "type": "attack", "base_damage": 50, "hits":[0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5], "next_skill": "finito_frecia"},
    "finito_frecia": {"name": "フィニトラ・フレティア", "type": "attack", "base_damage": 150, "hits": [1.0]},
    "zyuniouhoupai_daishahei": {"name": "十二王方牌大車併", "type": "attack", "base_damage": 50, "hits":[0.25,0.25,0.25,0.25,0.25,0.25,0.25,0.25,0.25,0.25,0.25,0.25],"next_skill": "kizan_shoukozin"},
    "kizan_shoukozin": {"name": "帰山笑紅塵", "type": "attack", "base_damage": 50, "hits":[1.0]},
    "sekiha_tenkyoken": {"name": "石破天驚拳", "type": "attack", "base_damage": 200, "hits":[1.0]},
    "kahun_dango": {"name": "かふんだんご", "type": "heal", "heal_amount": 3000} ,
    "cotton_guard": {"name": "コットンガード", "type": "buff", "buff_type": "defence", "buff_amount": 2.0, "duration": 3},
    "tenzyouno_inori": {"name": "天上の祈り", "type": "heal", "heal_amount": 200000},
    "connect": {"name": "ダブルフィニトラ・フレティア:　\nまどか「ほむらちゃん！」\nほむら「まどか！」\n2人「コネクト！！」", "type": "attack", "base_damage": 90, "hits": [1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 1.0, 999]},
    "kiwami": {"name": "うれ～んかいな～", "type": "attack", "base_damage": 50, "hits":[0.5,0.5]},
    "kurono_shougeki": {"name": "黒の衝撃", "type": "attack", "base_damage": 50, "hits":[1.0]} ,
    "shuenno_arashi": {"name": "終焉の嵐", "type": "attack", "base_damage": 30, "hits": [0.2,0.2,0.2]},    
    "wawawa_wasuremono": {"name": "wawawa忘れ物", "type": "attack", "base_damage": 50, "hits":[1.0]},
    "rage_buff": {"name": "ワルプルギスの怒り", "type": "buff", "buff_amount": {"attack": 2.0, "defence": 2.0}, "duration": 999}
}


#キャラクター作成(ステータス流用)
players = [
    Player("ほむら", 99999, 1500, 1175, SKILLS["headshot"], SKILLS["shooting_star"]),
    Player("東方不敗マスターアジア", 80000, 2000, 1100, SKILLS["zyuniouhoupai_daishahei"], SKILLS["sekiha_tenkyoken"]),
    Player("ワタシラガ", 80000, 300, 1050, SKILLS["cotton_guard"], SKILLS["kahun_dango"])    
]

enemies = [
    Enemy("白石稔", 3502500, 220, 550, SKILLS["wawawa_wasuremono"], SKILLS["wawawa_wasuremono"]),
    Enemy("ワルプルギスの夜", 5000000, 330, 570, SKILLS["kurono_shougeki"], SKILLS["shuenno_arashi"]),
    Enemy("CCO,Makoto", 3512500, 230, 530, SKILLS["kiwami"], SKILLS["kiwami"])
]

# 戦闘開始！
turn = 1
while any(p.is_alive() for p in players) and any(e.is_alive() for e in enemies):
    battle_turn(turn, players, enemies)
    turn += 1

# 戦闘結果
if any(p.is_alive() for p in players):
    print("🏆 プレイヤーの勝利！")
else:
    print("💀 敵の勝利…")

# ログを表示
for log in battle_log:
    print(log)
