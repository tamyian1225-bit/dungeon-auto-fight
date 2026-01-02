<!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8" />
<title>Dungeon Engine — Kernel Locked</title>
<meta name="viewport" content="width=device-width, initial-scale=1" />

<!-- 🔒 样式可空，但位置必须存在（允许你以后加） -->
<style>
/* UI / Canvas / Panel 的样式以后加
   Kernel 阶段不写任何布局 */
</style>
</head>

<body>

<!-- =====================================================
     🔒【KERNEL】完全冻结核心（禁止写玩法）
     职责：
     - 事件派发
     - 状态存储
     - 模块注册
     - 生命周期
===================================================== -->
<script>
"use strict";

/* =====================================================
   🔒 EventBus —— Kernel 级通信层
   - Kernel 不知道事件含义
   - 只负责派发
===================================================== */
const EventBus = (() => {
  const map = Object.create(null);

  return Object.freeze({
    on(event, handler){
      (map[event] ??= []).push(handler);
    },
    emit(event, payload){
      (map[event] || []).forEach(fn => fn(payload));
    }
  });
})();

/* =====================================================
   🔒 StateStore —— Kernel 级状态仓库
   - 只存数据
   - 不存逻辑
===================================================== */
const StateStore = (() => {
  const state = Object.create(null);

  return Object.freeze({
    set(key, value){
      state[key] = value;
    },
    get(key){
      return state[key];
    },
    snapshot(){
      return JSON.parse(JSON.stringify(state));
    }
  });
})();

/* =====================================================
   🔒 ModuleRegistry —— 模块注册器
   - Kernel 唯一认识模块的地方
   - 模块唯一入口 = init()
===================================================== */
const ModuleRegistry = (() => {
  const modules = [];

  return Object.freeze({
    register(module){
      if(!module || typeof module.init !== "function"){
        throw new Error("Invalid module: missing init()");
      }
      modules.push(module);
    },

    initAll(){
      modules.forEach(m => m.init());
    }
  });
})();

/* =====================================================
   🔒 ENGINE_BOOT —— Kernel 启动信号
   - Kernel 生命周期到此为止
===================================================== */
function ENGINE_BOOT(){
  console.log("[KERNEL] boot");
  ModuleRegistry.initAll();
  EventBus.emit("engine.boot");
}

Object.freeze(ENGINE_BOOT);

/*====开始====*/
const GameStartModule = {
  init(){
    EventBus.on("engine.boot", ()=>{
      // 初始化游戏状态
      StateStore.set("game.running", true);
      StateStore.set("game.startTime", Date.now());
      StateStore.set("game.frame", 0);

      // 对外广播：游戏已开始
      EventBus.emit("game.start");

      console.log("[GameStartModule] game.start");
    });
  }
};

ModuleRegistry.register(GameStartModule);

/*====循环====*/
const GameTickModule = {
  init(){
    EventBus.on("engine.boot", ()=>{
      let running = true;
      let lastTime = performance.now();

      const loop = (now)=>{
        if(!running) return;

        const dt = now - lastTime;
        lastTime = now;

        // 帧计数（纯数据）
        StateStore.set(
          "game.frame",
          (StateStore.get("game.frame") || 0) + 1
        );

        // 对外广播 Tick
        EventBus.emit("game.tick", dt);

        requestAnimationFrame(loop);
      };

      requestAnimationFrame(loop);

      // 监听结束信号（自我终止）
      EventBus.on("game.end", ()=>{
        running = false;
        console.log("[GameTickModule] stopped");
      });

      console.log("[GameTickModule] started");
    });
  }
};

ModuleRegistry.register(GameTickModule);

/*====结束====*/
const GameEndModule = {
  init(){
    EventBus.on("game.end", (reason)=>{
      StateStore.set("game.running", false);
      StateStore.set("game.endTime", Date.now());
      StateStore.set("game.endReason", reason || "unknown");

      console.log("[GameEndModule] game.end", reason);
    });
  }
};

ModuleRegistry.register(GameEndModule);

/* =====================================================
   [1.0.0] GLOBAL STAT RULES
   - 只定义规则
   - 不执行计算
===================================================== */
const GLOBAL_STAT_RULES = Object.freeze({

  // ===== 派生规则定义（公式，不执行）=====
  FORMULA: {
    ATK: "STR*3 + AGI*1",
    DEF: "DEX*2",
    HP:  "VIT*20",
    ASPD: "AGI -> interval reduce 1% per AGI, max 4 hits/sec",
    CR:  "CTR*1%",
    CRD: "CTR*50%"
  },

  // ===== 攻速规则 =====
  ASPD_RULE: {
    baseInterval: 1000,     // ms
    reducePerAGI: 0.01,     // 每点 AGI 减少 1%
    maxAttackPerSecond: 4  // 最多 4 次 / 秒
  }

});
/* =====================================================
   [1.0.1] PLAYER BASE STATS (START)
===================================================== */
const PLAYER_BASE_STATS = Object.freeze({

  base: {
    ATK: 20,
    DEF: 15,
    HP:  100,
    CR:  0,     // %
    CRD: 50     // %
  },

  // ===== 起始属性点 =====
  startStatPoint: 10,

  // ===== 起始等级与经验 =====
  level: 1,
  exp: 0,
  expNeed: 100   // 起始所需经验（你问过，这里是合理位置）
});
/* =====================================================
   [1.0.2] ATTRIBUTE → STAT CALC RULES
   - 只定义“点数如何影响数值”
===================================================== */
const ATTRIBUTE_CALC_RULES = Object.freeze({

  // ===== 属性点定义 =====
  ATTRIBUTES: ["STR","DEX","VIT","AGI","CTR"],

  // ===== 点数演算规则 =====
  EFFECT: {
    STR: {
      ATK: 3
    },
    AGI: {
      ATK: 1,
      ASPD: "see GLOBAL_STAT_RULES.ASPD_RULE"
    },
    DEX: {
      DEF: 2
    },
    VIT: {
      HP: 20
    },
    CTR: {
      CR: 1,     // %
      CRD: 50    // %
    }
  }

});
/* =====================================================
   [1.0.3] ENEMY BASE STAT TABLE
   - 玩家 & 怪物用同一套属性规则
===================================================== */
const ENEMY_BASE_STATS = Object.freeze({

  normal: {
    STR: 10,
    DEX: 10,
    VIT: 10,
    AGI: 10,
    CTR: 5
  },

  boss: {
    STR: 20,
    DEX: 10,
    VIT: 20,
    AGI: 10,
    CTR: 10
  }

});
/* =====================================================
   [1.0.4] EQUIPMENT BASE STAT TABLE
   - 装备演化在装备系统
   - 这里只定义“基础数值含义”
===================================================== */
const EQUIPMENT_BASE_STATS = Object.freeze({

  weapon: {
    ATK: [5, 10]
  },

  armor: {
    DEF: [3, 8]
  },

  helmet: {
    HP: [20, 50]
  },

  accessory: {
    CR: [1, 5]
  }

});

/* =====================================================
   [2.0.0] PlayerEntityModule
   - 玩家“实体状态”定义
   - ❌ 不做任何数值计算
   - ❌ 不依赖 1.0 DataMaster
   - ✅ 只负责 player 这个对象本身
===================================================== */
const PlayerEntityModule = {
  init(){

    EventBus.on("engine.boot", ()=>{

      /* 🔹 Player Entity（唯一玩家实体） */
      const player = {
        id: "player_001",

        /* ===== 基础身份 ===== */
        level: 1,
        class: null,          // 31 级后才会赋值
        statPoint: 10,        // 起始属性点（UI 后续操作）

        /* ===== 技能（只存 ID，不存数值） ===== */
        skills: [
          // 例："warrior_passive_01"
        ],

        /* ===== Runtime 状态 ===== */
        runtime: {
          hp: 0,              // 实际当前血量
          alive: true,
          status: []          // 中毒 / 灼烧 / 冻结等（字符串或对象）
        }
      };

      StateStore.set("player.entity", player);

      console.log("[2.0.0 PlayerEntityModule] player.entity initialized");
    });

  }
};

ModuleRegistry.register(PlayerEntityModule);
/* =====================================================
   [2.0.1] PlayerRuntimeModule
   - 只维护 runtime 状态
   - 不关心数值从哪里来
===================================================== */
const PlayerRuntimeModule = {
  init(){

    /* 初始化 HP（由 Calculator 或外部系统设置） */
    EventBus.on("player.runtime.init", ({ hp })=>{
      const player = StateStore.get("player.entity");
      if(!player) return;

      player.runtime.hp = hp;
      player.runtime.alive = true;

      StateStore.set("player.entity", player);
      EventBus.emit("player.runtime.update", player.runtime);
    });

    /* 扣血 */
    EventBus.on("player.damage", ({ damage })=>{
      const player = StateStore.get("player.entity");
      if(!player || !player.runtime.alive) return;

      player.runtime.hp -= damage;

      if(player.runtime.hp <= 0){
        player.runtime.hp = 0;
        player.runtime.alive = false;
        EventBus.emit("player.dead");
      }

      StateStore.set("player.entity", player);
      EventBus.emit("player.runtime.update", player.runtime);
    });

  }
};

ModuleRegistry.register(PlayerRuntimeModule);
/* =====================================================
   [2.0.2] PlayerLevelModule
   - 等级 / 属性点变化
   - ❌ 不计算属性
===================================================== */
const PlayerLevelModule = {
  init(){

    EventBus.on("player.level.up", ()=>{
      const player = StateStore.get("player.entity");
      if(!player) return;

      player.level += 1;
      player.statPoint += 5; // 每级给多少点（规则可以后改）

      StateStore.set("player.entity", player);

      EventBus.emit("player.level.changed", {
        level: player.level,
        statPoint: player.statPoint
      });
    });

  }
};

ModuleRegistry.register(PlayerLevelModule);
/* =====================================================
   [2.0.3] PlayerDeathModule
   - 玩家死亡事件
===================================================== */
const PlayerDeathModule = {
  init(){

    EventBus.on("player.dead", ()=>{
      const player = StateStore.get("player.entity");
      if(!player) return;

      player.runtime.alive = false;
      StateStore.set("player.entity", player);

      console.log("[PlayerDeathModule] player dead");
      EventBus.emit("game.end", "player_dead");
    });

  }
};

ModuleRegistry.register(PlayerDeathModule);
/* ---------- 2.1.0 EnemyEntityModule ---------- */
const EnemyEntityModule = {
  init() {

    EventBus.on("engine.boot", ()=>{

      /* 🔹 Enemy Entity（唯一敌人实体） */
      const normalEnemy = {
        id: "enemy_001",      // 敌人唯一ID
        type: "normal",       // 敌人类型（normal / boss）
        level: 1,             // 敌人等级
        runtime: {            // 敌人动态数据
          hp: 100,            // 当前血量
          alive: true,        // 敌人是否存活
          status: "normal"    // 敌人状态（normal / dead 等）
        }
      };

      const bossEnemy = {
        id: "boss_001",       // Boss 唯一ID
        type: "boss",         // Boss 类型
        level: 10,            // Boss 等级
        runtime: {            // Boss 动态数据
          hp: 500,            // 当前血量
          alive: true,        // Boss 是否存活
          status: "normal"    // Boss 状态
        }
      };

      // 保存敌人数据
      StateStore.set("enemy.normal", normalEnemy);
      StateStore.set("enemy.boss", bossEnemy);

      console.log("[2.1.0 EnemyEntityModule] enemy entity initialized.");
    });
  }
};

ModuleRegistry.register(EnemyEntityModule);
/* ---------- 2.1.1 EnemyRuntimeModule ---------- */
const EnemyRuntimeModule = {
  init(){

    /* 初始化敌人血量（由外部系统设置） */
    EventBus.on("enemy.runtime.init", ({ id, hp })=>{
      const enemy = StateStore.get(`enemy.${id}`);
      if(!enemy) return;

      enemy.runtime.hp = hp;
      enemy.runtime.alive = true;

      StateStore.set(`enemy.${id}`, enemy);
      EventBus.emit("enemy.runtime.update", enemy.runtime);
    });

    /* 敌人受到伤害 */
    EventBus.on("enemy.damage", ({ id, damage })=>{
      const enemy = StateStore.get(`enemy.${id}`);
      if(!enemy || !enemy.runtime.alive) return;

      enemy.runtime.hp -= damage;

      if(enemy.runtime.hp <= 0){
        enemy.runtime.hp = 0;
        enemy.runtime.alive = false;
        EventBus.emit("enemy.dead", enemy);  // 敌人死亡时广播
      }

      StateStore.set(`enemy.${id}`, enemy);
      EventBus.emit("enemy.runtime.update", enemy.runtime);
    });

  }
};

ModuleRegistry.register(EnemyRuntimeModule);
/* ---------- 2.1.3 EnemyLevelModule ---------- */
const EnemyLevelModule = {
  init() {

    // 监听敌人等级变化
    EventBus.on("enemy.level.up", ({ id }) => {
      const enemy = StateStore.get(`enemy.${id}`);
      if(!enemy) return;

      // 等级提升后的基础属性增幅
      enemy.level += 1;
      enemy.runtime.hp += 20;  // 例：每级增加 20 HP

      // 更新敌人数据
      StateStore.set(`enemy.${id}`, enemy);
      EventBus.emit("enemy.level.changed", { id: enemy.id, level: enemy.level });
    });

  }
};

ModuleRegistry.register(EnemyLevelModule);

/* =====================================================
   [3.0.0] AttributeToStatCalculator
   - 输入：属性点
   - 输出：基础派生数值
   - ❌ 不写 StateStore
===================================================== */
const AttributeToStatCalculator = Object.freeze({

  calc(attr){
    const rules = ATTRIBUTE_CALC_RULES;
    const aspdRule = GLOBAL_STAT_RULES.ASPD_RULE;

    const STR = attr.STR || 0;
    const DEX = attr.DEX || 0;
    const VIT = attr.VIT || 0;
    const AGI = attr.AGI || 0;
    const CTR = attr.CTR || 0;

    const ATK = STR * 3 + AGI * 1;
    const DEF = DEX * 2;
    const HP  = VIT * 20;

    const interval =
      aspdRule.baseInterval *
      Math.max(
        1 - AGI * aspdRule.reducePerAGI,
        1 / aspdRule.maxAttackPerSecond
      );

    const CR  = CTR * 1;     // %
    const CRD = CTR * 50;    // %

    return {
      ATK,
      DEF,
      HP,
      ASPD: interval,
      CR,
      CRD
    };
  }

});
/* =====================================================
   [3.0.1] PlayerStatCalculator
   - 只算玩家基础战斗数值
===================================================== */
const PlayerStatCalculator = Object.freeze({

  calc(playerAttr){
    const base = PLAYER_BASE_STATS.base;

    const derived = AttributeToStatCalculator.calc(playerAttr);

    return {
      ATK: base.ATK + derived.ATK,
      DEF: base.DEF + derived.DEF,
      HP:  base.HP  + derived.HP,
      ASPD: derived.ASPD,
      CR:  base.CR  + derived.CR,
      CRD: base.CRD + derived.CRD
    };
  }

});
/* =====================================================
   [3.0.2] EnemyStatCalculator
   - normal / boss 都走同一套规则
===================================================== */
const EnemyStatCalculator = Object.freeze({

  calc(type){
    const baseAttr = ENEMY_BASE_STATS[type];
    if(!baseAttr) throw new Error("Invalid enemy type");

    return AttributeToStatCalculator.calc(baseAttr);
  }

});
/* =====================================================
   [3.0.3] EquipmentBonusCalculator
   - 输入：当前数值 + 装备列表
   - 输出：加成后的数值
===================================================== */
const EquipmentBonusCalculator = Object.freeze({

  apply(baseStat, equipments){
    let result = { ...baseStat };

    equipments.forEach(eq=>{
      for(const k in eq.bonus){
        result[k] = (result[k] || 0) + eq.bonus[k];
      }
    });

    return result;
  }

});


/* ---------- 4.0.0 DamageCalcModule ---------- */
const DamageCalcModule = {
  init() {

    // 计算玩家攻击敌人时的伤害
    const calcDamage = (player, enemy, skill) => {
      const playerDerived = StateStore.get("player.derived");  // 获取玩家的派生属性
      const enemyDerived = StateStore.get(`enemy.${enemy.id}.derived`);  // 获取敌人的派生属性

      if(!playerDerived || !enemyDerived) return;

      // 计算基础伤害：玩家 ATK - 敌人 DEF
      let damage = playerDerived.ATK - enemyDerived.DEF;
      
      // 确保伤害不会小于1
      damage = Math.max(damage, 1);

      // 计算技能加成
      if (skill && skill.damageMultiplier) {
        damage *= skill.damageMultiplier;  // 如果技能有伤害增幅，加成到基础伤害
      }

      console.log(`[DamageCalcModule] Calculated damage: ${damage}`);

      // 计算暴击加成
      damage = this.calcCriticalDamage(damage, player);

      return damage;
    };

    // 计算暴击伤害
    const calcCriticalDamage = (damage, player) => {
      const playerDerived = StateStore.get("player.derived");

      if(!playerDerived) return damage;

      // 判断是否暴击
      const critChance = playerDerived.CR;
      const isCritical = Math.random() * 100 < critChance;

      if(isCritical) {
        const critDamage = damage * (1 + (playerDerived.CRD / 100));
        console.log(`[CriticalCalcModule] Critical hit! Damage: ${critDamage}`);
        return critDamage;  // 返回暴击伤害
      }

      console.log(`[CriticalCalcModule] No critical hit. Damage: ${damage}`);
      return damage;  // 非暴击时返回原始伤害
    };

    // 暴露给外部调用
    window.calcDamage = calcDamage;
    window.calcCriticalDamage = calcCriticalDamage;
  }
};

ModuleRegistry.register(DamageCalcModule);
/* ---------- 4.0.1 CombatTurnModule ---------- */
const CombatTurnModule = {
  init() {
    let playerActionTimer = 0;
    let enemyActionTimer = 0;

    // 每帧更新回合状态
    const updateCombatTurn = () => {
      const player = StateStore.get("player");
      const enemy = StateStore.get("enemy.normal");  // 默认普通敌人

      if(!player || !enemy) return;

      // 计算玩家行动间隔
      playerActionTimer += 1;
      if(playerActionTimer >= player.derived.ASPD) {
        playerActionTimer = 0;
        EventBus.emit("combat.player.attack", player, enemy);  // 玩家攻击事件
      }

      // 计算敌人行动间隔
      enemyActionTimer += 1;
      if(enemyActionTimer >= enemy.derived.ASPD) {
        enemyActionTimer = 0;
        EventBus.emit("combat.enemy.attack", player, enemy);  // 敌人攻击事件
      }
    };

    // 监听战斗开始
    EventBus.on("combat.start", () => {
      setInterval(updateCombatTurn, 1000 / 60);  // 每帧更新（假设 60fps）
    });
  }
};

ModuleRegistry.register(CombatTurnModule);
/* ---------- 4.0.2 CombatAttackModule ---------- */
const CombatAttackModule = {
  init() {

    // 玩家攻击敌人
    EventBus.on("combat.player.attack", (player, enemy) => {
      const damage = DamageCalcModule.calcDamage(player, enemy);  // 计算伤害
      const critDamage = DamageCalcModule.calcCriticalDamage(damage, player);  // 计算暴击伤害

      console.log(`[CombatAttackModule] Player attacks enemy. Damage: ${critDamage}`);

      // 执行伤害
      const enemyRuntime = StateStore.get(`enemy.${enemy.id}.runtime`);
      enemyRuntime.hp -= critDamage;

      if (enemyRuntime.hp <= 0) {
        enemyRuntime.hp = 0;
        enemyRuntime.alive = false;
        EventBus.emit("enemy.dead", enemy);  // 敌人死亡事件
      }

      StateStore.set(`enemy.${enemy.id}.runtime`, enemyRuntime);  // 更新敌人状态
    });

    // 敌人攻击玩家
    EventBus.on("combat.enemy.attack", (player, enemy) => {
      const damage = DamageCalcModule.calcDamage(enemy, player);  // 计算敌人攻击的伤害
      const critDamage = DamageCalcModule.calcCriticalDamage(damage, enemy);  // 计算暴击伤害

      console.log(`[CombatAttackModule] Enemy attacks player. Damage: ${critDamage}`);

      // 执行伤害
      const playerRuntime = StateStore.get("player.runtime");
      playerRuntime.hp -= critDamage;

      if (playerRuntime.hp <= 0) {
        playerRuntime.hp = 0;
        playerRuntime.alive = false;
        EventBus.emit("player.dead", player);  // 玩家死亡事件
      }

      StateStore.set("player.runtime", playerRuntime);  // 更新玩家状态
    });
  }
};

ModuleRegistry.register(CombatAttackModule);
/* ---------- 4.0.3 CombatEndModule ---------- */
const CombatEndModule = {
  init() {

    let combatActive = false;

    /* 战斗开始 */
    EventBus.on("combat.start", () => {
      combatActive = true;
      console.log("[4.0.3 CombatEndModule] combat started");
    });

    /* 玩家死亡 → 战斗失败 */
    EventBus.on("player.dead", () => {
      if (!combatActive) return;

      combatActive = false;

      console.log("[4.0.3 CombatEndModule] player defeated");

      EventBus.emit("combat.end", {
        result: "lose",
        reason: "player_dead"
      });
    });

    /* 敌人死亡 → 战斗胜利 */
    EventBus.on("enemy.dead", (enemy) => {
      if (!combatActive) return;

      combatActive = false;

      console.log("[4.0.3 CombatEndModule] enemy defeated", enemy.id);

      EventBus.emit("combat.end", {
        result: "win",
        enemyId: enemy.id,
        enemyType: enemy.type
      });
    });

    /* 战斗结束统一出口 */
    EventBus.on("combat.end", (payload) => {
      console.log("[4.0.3 CombatEndModule] combat end payload:", payload);

      /*
        这里只广播，不做：
        ❌ 掉落
        ❌ 经验
        ❌ UI
        ❌ 下一关
      */

      EventBus.emit("combat.cleanup", payload);
    });

  }
};

ModuleRegistry.register(CombatEndModule);
/* ---------- 5.0.0 SkillRegistryModule ---------- */
const SkillRegistryModule = {
  init() {

    // 注册技能
    const skills = [
      { id: "warrior_strike", damageMultiplier: 1.5, class: "warrior" },
      { id: "knight_shield_bash", damageMultiplier: 2, class: "knight" },
      // 更多技能...
    ];

    // 注册技能到玩家
    const player = StateStore.get("player");
    if (!player) return;

    player.skills = skills.filter(skill => skill.class === player.class);
    StateStore.set("player.skills", player.skills);

    console.log("[SkillRegistryModule] Skills registered:", player.skills);
  }
};

ModuleRegistry.register(SkillRegistryModule);
/* ---------- 5.0.0 ClassBonusModule ---------- */
const ClassBonusModule = {
  init() {
    EventBus.on("player.levelup", () => {
      const base = StateStore.get("player.base");
      const cls = StateStore.get("player.class");
      const skills = StateStore.get("player.skills");

      const add = (k, v) => base[k] = (base[k] || 0) + v;
      const give = (id) => { if (!skills.includes(id)) skills.push(id); };

      // 根据职业加成属性
      if (cls === "adventurer") {
        add("STR", 1); add("DEX", 1); add("VIT", 1);
      }

      if (cls === "warrior") {
        add("STR", 3); add("DEX", 3); add("VIT", 5);
        give("slash");
      }

      if (cls === "hunter") {
        add("STR", 3); add("AGI", 3); add("VIT", 2); add("CTR", 3);
        give("volley");
      }

      if (cls === "paladin") {
        add("STR", 5); add("DEX", 5); add("VIT", 10);
        give("holy_bless"); give("holy_light");
      }

      if (cls === "archer") {
        add("STR", 5); add("AGI", 5); add("VIT", 5); add("CTR", 5);
        give("eagle_eye"); give("stealth"); give("arrow_rain");
      }

      if (cls === "hero") {
        add("STR", 10); add("DEX", 10); add("VIT", 20); add("CTR", 10);
        give("hero_proof"); give("hero_pressure"); give("goddess_bless");
      }

      if (cls === "sharpshooter") {
        add("STR", 20); add("AGI", 10); add("DEX", 10); add("VIT", 10); add("CTR", 20);
        give("sharpshooter"); give("mana_loss"); give("god_step");
        give("insight"); give("double_arrow");
      }

      // 更新玩家的属性和技能
      StateStore.set("player.base", base);
      StateStore.set("player.skills", skills);
      EventBus.emit("player.stat.change");
    });
  }
};

ModuleRegistry.register(ClassBonusModule);
/* ---------- 5.0.1 SkillRegistryModule ---------- */
const SkillRegistryModule = {
  init() {
    // 注册技能数据
    const skillData = {
      slash: { id: "slash", type: "active", trigger: "onAttack", rate: 0.2, effect: "slash" },
      volley: { id: "volley", type: "active", trigger: "onAttack", rate: 0.2, effect: "volley" },
      holy_bless: { id: "holy_bless", type: "passive", effect: "stat_bonus", bonus: { VIT: 50, CTR: 50 } },
      holy_light: { id: "holy_light", type: "active", trigger: "onAttack", rate: 1.0, effect: "holy_light" },
      eagle_eye: { id: "eagle_eye", type: "active", effect: "buff", buff: { ATK: 10 } },
      stealth: { id: "stealth", type: "active", effect: "buff", buff: { AGI: 5 } },
      hero_proof: { id: "hero_proof", type: "passive", effect: "all_stat_percent", value: 0.5 },
      hero_pressure: { id: "hero_pressure", type: "passive", ignoreDef: 0.5, reduceDamage: 0.5 },
      goddess_bless: { id: "goddess_bless", type: "passive", finalDamageMul: 2 },
      sharpshooter: { id: "sharpshooter", type: "active", effect: "damageMultiplier", value: 2 },
      mana_loss: { id: "mana_loss", type: "debuff", effect: "manaDrain", rate: 0.1 }
    };

    // 更新技能数据到 StateStore
    StateStore.set("skill.registry", skillData);
    console.log("[SkillRegistryModule] Skills registered:", skillData);
  }
};

ModuleRegistry.register(SkillRegistryModule);
/* ---------- 5.0.2 SkillTriggerModule ---------- */
const SkillTriggerModule = {
  init() {
    // 监听玩家攻击触发技能
    EventBus.on("combat.player.attack", () => {
      const skills = StateStore.get("player.skills") || [];
      const skillRegistry = StateStore.get("skill.registry");

      skills.forEach(skillId => {
        const skill = skillRegistry[skillId];
        if (!skill || skill.trigger !== "onAttack") return;
        if (Math.random() > (skill.rate || 1)) return;

        // 触发技能
        EventBus.emit("skill.cast", { skillId: skill.id, source: "player", target: "enemy" });
      });
    });
  }
};

ModuleRegistry.register(SkillTriggerModule);
/* ---------- 5.0.3 SkillEffectModule ---------- */
const SkillEffectModule = {
  init() {
    EventBus.on("skill.context.ready", ({ skill, source, target }) => {
      if (!StateStore.get("combat.active")) return;

      const enemy = StateStore.get("enemy.base");
      if (!enemy || !enemy.alive) return;

      let damage = skill.damage || 0;

      // 计算技能效果（例如：斩击、箭雨、圣光等）
      if (skill.effect === "slash") {
        damage *= 1.5; // 斩击技能的伤害加成
      } else if (skill.effect === "volley") {
        damage *= 2; // 箭雨技能的伤害加成
      } else if (skill.effect === "holy_light") {
        damage *= 2; // 圣光技能的伤害加成
      }

      // 应用伤害或增益/减益效果
      EventBus.emit("combat.damage.final", {
        damage: Math.floor(damage),
        target: "enemy",
        crit: false
      });
    });
  }
};

ModuleRegistry.register(SkillEffectModule);
/* =====================================================
   [5.1.0] EnemyExpTable
   - 只定义“击杀给多少经验”
===================================================== */
const ENEMY_EXP_TABLE = Object.freeze({
  normal: level => 20 + level * 5,
  boss:   level => 200 + level * 50
});
/* =====================================================
   [5.1.1] ExpGainModule
   - 只负责：敌人死亡 → 发经验
===================================================== */
const ExpGainModule = {
  init(){

    EventBus.on("enemy.dead", (enemy)=>{

      const player = StateStore.get("player.entity");
      if(!player) return;

      const expGain = ENEMY_EXP_TABLE[enemy.type]?.(enemy.level) ?? 0;

      EventBus.emit("player.exp.gain", {
        value: expGain,
        source: enemy.id
      });

      console.log("[ExpGainModule] gain exp:", expGain);
    });

  }
};
ModuleRegistry.register(ExpGainModule);
/* =====================================================
   [5.1.2] PlayerExpModule
   - 只做 exp 累加
===================================================== */
const PlayerExpModule = {
  init(){

    EventBus.on("player.exp.gain", ({ value })=>{
      const player = StateStore.get("player.entity");
      if(!player) return;

      player.exp = (player.exp || 0) + value;

      StateStore.set("player.entity", player);

      EventBus.emit("player.exp.changed", {
        exp: player.exp
      });
    });

  }
};
ModuleRegistry.register(PlayerExpModule);
/* =====================================================
   [5.1.3] PlayerLevelCheckModule
   - 只判断是否升级
===================================================== */
const PlayerLevelCheckModule = {
  init(){

    EventBus.on("player.exp.changed", ()=>{
      const player = StateStore.get("player.entity");
      if(!player) return;

      const need = PLAYER_BASE_STATS.expNeed;

      while(player.exp >= need){
        player.exp -= need;
        EventBus.emit("player.level.up");
      }

      StateStore.set("player.entity", player);
    });

  }
};
ModuleRegistry.register(PlayerLevelCheckModule);

/* ---------- 6.0.0 EquipmentModule ---------- */
const EquipmentModule = {
  init() {
    const player = StateStore.get("player");
    if (!player) return;

    player.equipment = {
      weapon: null,
      armor: null,
      accessory: null
    };

    StateStore.set("player.equipment", player.equipment);
  },

  equipItem(item) {
    const player = StateStore.get("player");
    const equipment = player.equipment;

    if (item.type === "weapon") {
      equipment.weapon = item;
    } else if (item.type === "armor") {
      equipment.armor = item;
    } else if (item.type === "accessory") {
      equipment.accessory = item;
    }

    this.removeItemFromInventory(item);
    StateStore.set("player.equipment", equipment);
    this.applyEquipmentBonuses(player, item);
    EventBus.emit("player.equipment.change", equipment);
  },

  unequipItem(item) {
    const player = StateStore.get("player");
    const equipment = player.equipment;

    if (item === equipment.weapon) {
      equipment.weapon = null;
    } else if (item === equipment.armor) {
      equipment.armor = null;
    } else if (item === equipment.accessory) {
      equipment.accessory = null;
    }

    this.removeEquipmentBonuses(player, item);
    this.addItemToInventory(item);
    StateStore.set("player.equipment", equipment);
    EventBus.emit("player.equipment.change", equipment);
  },

  applyEquipmentBonuses(player, item) {
    let derived = { ...player.derived };
    if (item.type === "weapon") {
      derived.ATK += item.bonus.ATK || 0;
    } else if (item.type === "armor") {
      derived.DEF += item.bonus.DEF || 0;
    } else if (item.type === "accessory") {
      derived.CTR += item.bonus.CTR || 0;
    }

    StateStore.set("player.derived", derived);
  },

  removeEquipmentBonuses(player, item) {
    let derived = { ...player.derived };
    if (item.type === "weapon") {
      derived.ATK -= item.bonus.ATK || 0;
    } else if (item.type === "armor") {
      derived.DEF -= item.bonus.DEF || 0;
    } else if (item.type === "accessory") {
      derived.CTR -= item.bonus.CTR || 0;
    }

    StateStore.set("player.derived", derived);
  },

  removeItemFromInventory(item) {
    const player = StateStore.get("player");
    if (!player.inventory) return;

    const index = player.inventory.findIndex(i => i.name === item.name);
    if (index >= 0) {
      player.inventory.splice(index, 1);
      StateStore.set("player.inventory", player.inventory);
    }
  },

  addItemToInventory(item) {
    const player = StateStore.get("player");
    if (!player.inventory) player.inventory = [];

    player.inventory.push(item);
    StateStore.set("player.inventory", player.inventory);
  }
};

ModuleRegistry.register(EquipmentModule);
/* ---------- 6.1 ItemGenerateModule ---------- */
const ItemGenerateModule = {
  init() {
    const droppedItem = this.generateItem();
    this.addItemToInventory(droppedItem);
  },

  generateItem() {
    let item = {};
    const random = Math.random();
    if (random < 0.5) {
      item = { name: "Common Sword", type: "weapon", bonus: { ATK: 5 }, quality: "common" };
    } else if (random < 0.8) {
      item = { name: "Rare Sword", type: "weapon", bonus: { ATK: 10 }, quality: "rare" };
    } else {
      item = { name: "Epic Sword", type: "weapon", bonus: { ATK: 20 }, quality: "epic" };
    }
    return item;
  },

  addItemToInventory(item) {
    const player = StateStore.get("player");
    if (!player.inventory) player.inventory = [];

    player.inventory.push(item);
    StateStore.set("player.inventory", player.inventory);
  }
};

ModuleRegistry.register(ItemGenerateModule);
/* ---------- 6.2 InventoryModule ---------- */
const InventoryModule = {
  init() {
    const player = StateStore.get("player");
    if (!player.inventory) player.inventory = [];
    StateStore.set("player.inventory", player.inventory);
  },

  addItemToInventory(item) {
    const player = StateStore.get("player");
    if (!player.inventory) player.inventory = [];

    player.inventory.push(item);
    StateStore.set("player.inventory", player.inventory);
  },

  removeItemFromInventory(item) {
    const player = StateStore.get("player");
    if (!player.inventory) return;

    const index = player.inventory.findIndex(i => i.name === item.name);
    if (index >= 0) {
      player.inventory.splice(index, 1);
      StateStore.set("player.inventory", player.inventory);
    }
  },

  discardItem(item) {
    this.removeItemFromInventory(item);
  }
};

ModuleRegistry.register(InventoryModule);
/* ---------- 7.0.0 UIKernelModule ---------- */
/*
职责：
- 统一 UI 刷新入口
- 所有 UI 模块只监听 UI 事件
*/
const UIKernelModule = {
  init(){
    EventBus.on("ui.refresh", (scope)=>{
      // scope: "player" | "enemy" | "inventory" | "combat" | "all"
    });
  }
};
ModuleRegistry.register(UIKernelModule);
/* ---------- 7.0.1 PlayerPanelUIModule ---------- */
/*
显示：
- HP / MAX HP
- ATK / DEF / ASPD / CR / CRD
- 当前职业
*/
const PlayerPanelUIModule = {
  init(){
    EventBus.on("player.stat.change", ()=> this.render());
    EventBus.on("player.hp.change", ()=> this.render());
  },
  render(){
    const p = StateStore.get("player");
    const d = StateStore.get("player.derived");
    // ⚠️ 这里只读，不计算
  }
};
ModuleRegistry.register(PlayerPanelUIModule);
/* ---------- 7.0.2 EnemyPanelUIModule ---------- */
const EnemyPanelUIModule = {
  init(){
    EventBus.on("enemy.spawn", ()=> this.render());
    EventBus.on("enemy.hp.change", ()=> this.render());
  },
  render(){
    const e = StateStore.get("enemy");
  }
};
ModuleRegistry.register(EnemyPanelUIModule);
/* ---------- 7.0.3 InventoryUIModule ---------- */
/*
显示：
- 背包物品列表
- 装备 / 丢弃按钮
*/
const InventoryUIModule = {
  init(){
    EventBus.on("inventory.change", ()=> this.render());
  },
  render(){
    const inv = StateStore.get("player.inventory");
  }
};
ModuleRegistry.register(InventoryUIModule);
/* ---------- 7.0.4 EquipmentUIModule ---------- */
/*
显示：
- weapon / armor / accessory
*/
const EquipmentUIModule = {
  init(){
    EventBus.on("player.equipment.change", ()=> this.render());
  },
  render(){
    const eq = StateStore.get("player.equipment");
  }
};
ModuleRegistry.register(EquipmentUIModule);
/* ---------- 7.0.5 CombatLogUIModule ---------- */
/*
显示：
- 攻击
- 伤害
- 暴击
- 技能触发
*/
const CombatLogUIModule = {
  init(){
    EventBus.on("combat.log", msg => this.push(msg));
  },
  push(msg){}
};
ModuleRegistry.register(CombatLogUIModule);
/* ---------- 8.0.0 AnimationKernelModule ---------- */
const AnimationKernelModule = {
  init(){
    EventBus.on("animation.play", payload=>{
      // payload: { type, source, target }
    });
  }
};
ModuleRegistry.register(AnimationKernelModule);
/* ---------- 8.0.1 AttackAnimationModule ---------- */
const AttackAnimationModule = {
  init(){
    EventBus.on("combat.attack", ({source,target})=>{
      EventBus.emit("animation.play", {
        type:"attack",
        source,
        target
      });
    });
  }
};
ModuleRegistry.register(AttackAnimationModule);
/* ---------- 8.0.2 DamageNumberAnimationModule ---------- */
const DamageNumberAnimationModule = {
  init(){
    EventBus.on("combat.damage.final", ({damage,crit,target})=>{
      EventBus.emit("animation.play",{
        type:"damageNumber",
        value:damage,
        crit,
        target
      });
    });
  }
};
ModuleRegistry.register(DamageNumberAnimationModule);
/* ---------- 8.0.3 DeathAnimationModule ---------- */
const DeathAnimationModule = {
  init(){
    EventBus.on("enemy.dead", ()=> {
      EventBus.emit("animation.play",{type:"enemyDead"});
    });
    EventBus.on("player.dead", ()=> {
      EventBus.emit("animation.play",{type:"playerDead"});
    });
  }
};
ModuleRegistry.register(DeathAnimationModule);

/* =====================================================
   [9.0.0] UIStateBinderModule
   - StateStore → UI Snapshot
===================================================== */
const UIStateBinderModule = {
  init(){

    const sync = ()=>{
      StateStore.set("ui.snapshot", {
        player: StateStore.get("player.entity"),
        playerDerived: StateStore.get("player.derived"),
        enemyNormal: StateStore.get("enemy.normal"),
        enemyBoss: StateStore.get("enemy.boss"),
        inventory: StateStore.get("player.inventory"),
        equipment: StateStore.get("player.equipment"),
        combat: {
          running: StateStore.get("combat.active")
        }
      });

      EventBus.emit("ui.refresh", "all");
    };

    EventBus.on("player.runtime.update", sync);
    EventBus.on("player.stat.change", sync);
    EventBus.on("enemy.runtime.update", sync);
    EventBus.on("inventory.change", sync);
    EventBus.on("player.equipment.change", sync);
    EventBus.on("combat.start", sync);
    EventBus.on("combat.end", sync);
  }
};

ModuleRegistry.register(UIStateBinderModule);
/* =====================================================
   [9.0.1] PlayerUIRenderModule
===================================================== */
const PlayerUIRenderModule = {
  init(){
    EventBus.on("ui.refresh", scope=>{
      if(scope==="all" || scope==="player"){
        this.render();
      }
    });
  },

  render(){
    const snap = StateStore.get("ui.snapshot");
    if(!snap) return;

    const p = snap.player;
    const d = snap.playerDerived;

    console.log("[UI][Player]",
      `HP ${p.runtime.hp}`,
      `ATK ${d?.ATK}`,
      `DEF ${d?.DEF}`,
      `CR ${d?.CR}%`,
      `CRD ${d?.CRD}%`
    );
  }
};

ModuleRegistry.register(PlayerUIRenderModule);
/* =====================================================
   [9.0.2] EnemyUIRenderModule
===================================================== */
const EnemyUIRenderModule = {
  init(){
    EventBus.on("ui.refresh", scope=>{
      if(scope==="all" || scope==="enemy"){
        this.render();
      }
    });
  },

  render(){
    const snap = StateStore.get("ui.snapshot");
    if(!snap) return;

    const e = snap.enemyNormal;
    if(!e) return;

    console.log("[UI][Enemy]",
      `HP ${e.runtime.hp}`,
      `Alive ${e.runtime.alive}`
    );
  }
};

ModuleRegistry.register(EnemyUIRenderModule);
/* =====================================================
   [9.0.3] InventoryUIRenderModule
===================================================== */
const InventoryUIRenderModule = {
  init(){
    EventBus.on("ui.refresh", scope=>{
      if(scope==="all" || scope==="inventory"){
        this.render();
      }
    });
  },

  render(){
    const snap = StateStore.get("ui.snapshot");
    if(!snap) return;

    console.log("[UI][Inventory]", snap.inventory);
  }
};

ModuleRegistry.register(InventoryUIRenderModule);
/* =====================================================
   [9.0.4] EquipmentUIRenderModule
===================================================== */
const EquipmentUIRenderModule = {
  init(){
    EventBus.on("ui.refresh", scope=>{
      if(scope==="all" || scope==="equipment"){
        this.render();
      }
    });
  },

  render(){
    const snap = StateStore.get("ui.snapshot");
    if(!snap) return;

    console.log("[UI][Equipment]", snap.equipment);
  }
};

ModuleRegistry.register(EquipmentUIRenderModule);
/* =====================================================
   [10.0.0] StageFlowModule
   - 职责：流程推进（关卡 / 层数）
   - ❌ 不生成敌人
   - ❌ 不计算数值
   - ❌ 不做 UI
===================================================== */
const StageFlowModule = {
  init(){

    /* ---------- 初始化 Stage ---------- */
    EventBus.on("game.start", ()=>{

      StateStore.set("stage.index", 1);
      StateStore.set("stage.state", "idle");
      StateStore.set("stage.lastResult", null);

      console.log("[10.0.0 StageFlowModule] stage initialized");
    });

    /* ---------- 战斗开始 → 锁定流程 ---------- */
    EventBus.on("combat.start", ()=>{

      StateStore.set("stage.state", "combat");

      console.log("[10.0.0 StageFlowModule] stage state = combat");
    });

    /* ---------- 战斗结束 ---------- */
    EventBus.on("combat.end", (payload)=>{

      StateStore.set("stage.state", "cleared");
      StateStore.set("stage.lastResult", payload.result);

      console.log(
        "[10.0.0 StageFlowModule] combat ended:",
        payload.result
      );

      // ❗这里只记录结果，不推进
    });

    /* ---------- 推进下一层（显式调用） ---------- */
    EventBus.on("stage.next", ()=>{

      const current = StateStore.get("stage.index") || 1;

      StateStore.set("stage.index", current + 1);
      StateStore.set("stage.state", "idle");
      StateStore.set("stage.lastResult", null);

      console.log(
        "[10.0.0 StageFlowModule] move to stage",
        current + 1
      );

      /*
        ⚠️ 注意：
        - 这里不 spawn enemy
        - 不 start combat
        - 只发信号
      */
      EventBus.emit("stage.enter", {
        index: current + 1
      });
    });

  }
};

ModuleRegistry.register(StageFlowModule);

/* =====================================================
   [10.0.1] StageConfigModule
   - 定义关卡配置
   - ❌ 不生成敌人
===================================================== */
const StageConfigModule = {
  init(){

    const STAGE_TABLE = {
      1: { type:"normal", enemyPool:"normal", count:1 },
      2: { type:"normal", enemyPool:"normal", count:1 },
      5: { type:"elite",  enemyPool:"elite",  count:1 },
      10:{ type:"boss",   enemyPool:"boss",   count:1 }
    };

    StateStore.set("stage.config.table", STAGE_TABLE);

    EventBus.on("stage.enter", ({ index })=>{
      const cfg = STAGE_TABLE[index] || {
        type:"normal",
        enemyPool:"normal",
        count:1
      };

      StateStore.set("stage.current.config", cfg);

      EventBus.emit("stage.config.ready", {
        stage:index,
        config:cfg
      });

      console.log("[10.0.1] stage config ready", index, cfg);
    });

  }
};
ModuleRegistry.register(StageConfigModule);
/* =====================================================
   [10.0.2] StageEnemyRequestModule
   - 关卡 → 请求敌人
===================================================== */
const StageEnemyRequestModule = {
  init(){

    EventBus.on("stage.config.ready", ({ stage, config })=>{

      EventBus.emit("enemy.request.spawn", {
        stage,
        pool: config.enemyPool,
        count: config.count
      });

      console.log("[10.0.2] enemy spawn requested");
    });

  }
};
ModuleRegistry.register(StageEnemyRequestModule);
/* =====================================================
   [10.0.3] StageCombatStartModule
===================================================== */
const StageCombatStartModule = {
  init(){

    EventBus.on("enemy.spawn.ready", ()=>{

      EventBus.emit("combat.start");
      console.log("[10.0.3] combat started by stage");
    });

  }
};
ModuleRegistry.register(StageCombatStartModule);
/* =====================================================
   [10.0.4] StageClearJudgeModule
===================================================== */
const StageClearJudgeModule = {
  init(){

    EventBus.on("enemy.dead", ()=>{

      const enemiesLeft = StateStore.get("enemy.alive.count") || 0;

      if(enemiesLeft <= 0){
        EventBus.emit("stage.cleared");
        console.log("[10.0.4] stage cleared");
      }
    });

  }
};
ModuleRegistry.register(StageClearJudgeModule);
/* =====================================================
   [10.0.5] StageRewardTriggerModule
===================================================== */
const StageRewardTriggerModule = {
  init(){

    EventBus.on("stage.cleared", ()=>{

      EventBus.emit("stage.reward.trigger", {
        stage: StateStore.get("stage.index")
      });

      console.log("[10.0.5] reward trigger emitted");
    });

  }
};
ModuleRegistry.register(StageRewardTriggerModule);
/* =====================================================
   [10.0.6] StageAdvanceModule
===================================================== */
const StageAdvanceModule = {
  init(){

    EventBus.on("stage.advance.confirm", ()=>{

      EventBus.emit("stage.next");

      console.log("[10.0.6] advance confirmed");
    });

  }
};
ModuleRegistry.register(StageAdvanceModule);
/* =====================================================
   [10.0.7] StageFailModule
===================================================== */
const StageFailModule = {
  init(){

    EventBus.on("player.dead", ()=>{

      StateStore.set("stage.state","failed");

      EventBus.emit("stage.failed");

      console.log("[10.0.7] stage failed");
    });

  }
};
ModuleRegistry.register(StageFailModule);

/* =====================================================
   [11.0.0] EnemyPoolModule
   - 定义敌人池
===================================================== */
const EnemyPoolModule = {
  init(){

    const POOLS = {
      normal: ["enemy_slime","enemy_goblin"],
      elite:  ["enemy_knight"],
      boss:   ["boss_dragon"]
    };

    StateStore.set("enemy.pool.table", POOLS);

    console.log("[11.0.0] enemy pool table ready");
  }
};

ModuleRegistry.register(EnemyPoolModule);
/* =====================================================
   [11.0.1] EnemySpawnModule
===================================================== */
const EnemySpawnModule = {
  init(){

    EventBus.on("enemy.request.spawn", ({ pool, count })=>{

      const table = StateStore.get("enemy.pool.table");
      const list = table[pool] || [];

      const enemies = [];

      for(let i=0;i<count;i++){
        const id = list[Math.floor(Math.random()*list.length)];

        enemies.push({
          uid: `${id}_${Date.now()}_${i}`,
          id,
          type: pool==="boss" ? "boss" : "normal",
          level: StateStore.get("stage.index") || 1,
          runtime:{
            hp:0,
            alive:true
          }
        });
      }

      StateStore.set("enemy.active.list", enemies);
      StateStore.set("enemy.alive.count", enemies.length);

      EventBus.emit("enemy.spawn.ready", enemies);

      console.log("[11.0.1] enemy spawned", enemies);
    });

  }
};

ModuleRegistry.register(EnemySpawnModule);
/* =====================================================
   [11.0.2] EnemyInitStatModule
===================================================== */
const EnemyInitStatModule = {
  init(){

    EventBus.on("enemy.spawn.ready", (enemies)=>{

      enemies.forEach(e=>{
        const derived = EnemyStatCalculator.calc(e.type);
        e.runtime.hp = derived.HP;
        e.runtime.maxHP = derived.HP;
      });

      StateStore.set("enemy.active.list", enemies);

      console.log("[11.0.2] enemy stat initialized");
    });

  }
};

ModuleRegistry.register(EnemyInitStatModule);
/* =====================================================
   [11.0.3] EnemyAITickModule
===================================================== */
const EnemyAITickModule = {
  init(){

    EventBus.on("game.tick", ()=>{

      if(StateStore.get("stage.state")!=="combat") return;

      const enemies = StateStore.get("enemy.active.list") || [];

      enemies.forEach(e=>{
        if(!e.runtime.alive) return;

        EventBus.emit("enemy.intent.attack", {
          enemy:e
        });
      });

    });

  }
};

ModuleRegistry.register(EnemyAITickModule);
/* =====================================================
   [11.0.4] EnemyAttackIntentModule
===================================================== */
const EnemyAttackIntentModule = {
  init(){

    EventBus.on("enemy.intent.attack", ({ enemy })=>{

      EventBus.emit("combat.enemy.attack", {
        source:enemy,
        target:"player"
      });

    });

  }
};

ModuleRegistry.register(EnemyAttackIntentModule);
/* =====================================================
   [11.0.5] EnemyDeathCountModule
===================================================== */
const EnemyDeathCountModule = {
  init(){

    EventBus.on("enemy.dead", ()=>{

      let n = StateStore.get("enemy.alive.count") || 0;
      n = Math.max(0, n-1);

      StateStore.set("enemy.alive.count", n);

      console.log("[11.0.5] enemy alive left:", n);
    });

  }
};

ModuleRegistry.register(EnemyDeathCountModule);
/* =====================================================
   [12.0.0] EnemyPoolTable
   - 普通怪池
===================================================== */
const ENEMY_POOL_TABLE = Object.freeze({

  normal: [
    { id:"slime",   base:"normal" },
    { id:"goblin",  base:"normal" },
    { id:"wolf",    base:"normal" }
  ],

  elite: [
    { id:"orc",     base:"normal" },
    { id:"ogre",    base:"normal" }
  ]

});
/* =====================================================
   [12.0.1] BossPoolTable
   - Boss 池
===================================================== */
const BOSS_POOL_TABLE = Object.freeze({

  boss: [
    { id:"dragon", base:"boss" },
    { id:"lich",   base:"boss" }
  ]

});
/* =====================================================
   [12.0.2] EnemyPoolSelectModule
   - 根据 pool 选模板
===================================================== */
const EnemyPoolSelectModule = {
  init(){

    EventBus.on("enemy.spawn.begin", ({ pool, count })=>{

      const poolTable =
        pool==="boss"
          ? BOSS_POOL_TABLE.boss
          : ENEMY_POOL_TABLE[pool] || ENEMY_POOL_TABLE.normal;

      const selected = [];

      for(let i=0;i<count;i++){
        const pick = poolTable[Math.floor(Math.random()*poolTable.length)];
        selected.push(pick);
      }

      StateStore.set("enemy.spawn.template", selected);

      console.log("[12.0.2] enemy template selected", selected);
    });

  }
};
ModuleRegistry.register(EnemyPoolSelectModule);
/* =====================================================
   [12.0.3] EnemyTemplateBindModule
   - 模板 → enemy entity
===================================================== */
const EnemyTemplateBindModule = {
  init(){

    EventBus.on("enemy.entity.created", (entities)=>{

      const templates = StateStore.get("enemy.spawn.template") || [];

      entities.forEach((e,i)=>{
        const tpl = templates[i];
        if(tpl){
          e.templateId = tpl.id;
          e.type = tpl.base;
        }
      });

      StateStore.set("enemy.active.list", entities);

      console.log("[12.0.3] enemy template bound");
    });

  }
};
ModuleRegistry.register(EnemyTemplateBindModule);
<!-- =====================================================
     [13.0.0] PixelKernelModule
     - 像素系统核心
     - 职责：
       - Canvas 创建
       - 像素缩放
       - 统一 draw 接口
===================================================== -->
<script>
const PixelKernelModule = {
  init(){

    const canvas = document.createElement("canvas");
    const ctx = canvas.getContext("2d");

    canvas.width  = 16 * 8;   // 默认显示 8 倍
    canvas.height = 16 * 8;
    canvas.style.imageRendering = "pixelated";
    canvas.style.border = "1px solid #333";

    document.body.appendChild(canvas);

    StateStore.set("pixel.canvas", canvas);
    StateStore.set("pixel.ctx", ctx);
    StateStore.set("pixel.scale", 8);

    EventBus.on("pixel.clear", ()=>{
      ctx.clearRect(0,0,canvas.width,canvas.height);
    });

    console.log("[13.0.0 PixelKernelModule] ready");
  }
};
ModuleRegistry.register(PixelKernelModule);
</script>

<!-- =====================================================
     [13.0.1] PixelPaletteModule
     - 像素调色板
===================================================== -->
<script>
const PixelPaletteModule = {
  init(){

    const PALETTE = {
      ".": "rgba(0,0,0,0)",

      "B": "#000000",
      "W": "#ffffff",
      "R": "#ff0000",
      "G": "#00ff00",
      "U": "#0000ff",

      "Y": "#ffff00",
      "O": "#ff9900",
      "P": "#cc66ff",
      "S": "#999999"
    };

    StateStore.set("pixel.palette", PALETTE);

    console.log("[13.0.1 PixelPaletteModule] palette loaded");
  }
};
ModuleRegistry.register(PixelPaletteModule);
</script>

<!-- =====================================================
     [13.0.2] PixelSpriteRegistryModule
     - 注册 16x16 像素模板
===================================================== -->
<script>
const PixelSpriteRegistryModule = {
  init(){

    const SPRITES = {

      player: [
        "................",
        "....BBBB........",
        "...BWWWWB.......",
        "..BW.WW.WB......",
        "..BWWWWWWB......",
        "...BBBBBB.......",
        "....B..B........",
        "...BB..BB.......",
        "..B........B....",
        "..B........B....",
        "..B........B....",
        "...B......B....",
        "....BBBBBB......",
        "................",
        "................",
        "................"
      ],

      slime: [
        "................",
        "................",
        "....GGGGGG......",
        "...GWWWWWWG.....",
        "..GWWWWWWWWG....",
        "..GWWWWWWWWG....",
        "..GWWWWWWWWG....",
        "...GGGGGGGG.....",
        "................",
        "................",
        "................",
        "................",
        "................",
        "................",
        "................",
        "................"
      ]

    };

    StateStore.set("pixel.sprite.table", SPRITES);

    console.log("[13.0.2 PixelSpriteRegistryModule] sprites registered");
  }
};
ModuleRegistry.register(PixelSpriteRegistryModule);
</script>

<!-- =====================================================
     [13.0.3] PixelDrawModule
     - 16x16 像素绘制
===================================================== -->
<script>
const PixelDrawModule = {
  init(){

    EventBus.on("pixel.draw", ({ spriteId, x=0, y=0 })=>{

      const ctx     = StateStore.get("pixel.ctx");
      const scale   = StateStore.get("pixel.scale");
      const palette = StateStore.get("pixel.palette");
      const table   = StateStore.get("pixel.sprite.table");

      const sprite = table[spriteId];
      if(!sprite) return;

      sprite.forEach((row, ry)=>{
        [...row].forEach((ch, rx)=>{
          const color = palette[ch];
          if(!color || ch===".") return;

          ctx.fillStyle = color;
          ctx.fillRect(
            (x + rx) * scale,
            (y + ry) * scale,
            scale,
            scale
          );
        });
      });

    });

    console.log("[13.0.3 PixelDrawModule] draw ready");
  }
};
ModuleRegistry.register(PixelDrawModule);
</script>

<!-- =====================================================
     [13.0.4] PlayerPixelRenderModule
     - 渲染玩家像素
===================================================== -->
<script>
const PlayerPixelRenderModule = {
  init(){

    EventBus.on("ui.refresh", ()=>{
      const player = StateStore.get("player.entity");
      if(!player || !player.runtime.alive) return;

      EventBus.emit("pixel.clear");
      EventBus.emit("pixel.draw", {
        spriteId:"player",
        x:0,
        y:0
      });
    });

    console.log("[13.0.4 PlayerPixelRenderModule] bound");
  }
};
ModuleRegistry.register(PlayerPixelRenderModule);
</script>

<!-- =====================================================
     [13.0.5] EnemyPixelRenderModule
     - 渲染敌人像素
===================================================== -->
<script>
const EnemyPixelRenderModule = {
  init(){

    EventBus.on("ui.refresh", ()=>{
      const enemies = StateStore.get("enemy.active.list") || [];
      if(enemies.length===0) return;

      enemies.forEach((e,i)=>{
        if(!e.runtime.alive) return;

        EventBus.emit("pixel.draw", {
          spriteId:"slime",
          x:20 + i*20,
          y:0
        });
      });
    });

    console.log("[13.0.5 EnemyPixelRenderModule] bound");
  }
};
ModuleRegistry.register(EnemyPixelRenderModule);
</script>

/* =====================================================
   [13.0.6] ScreenShakeModule
   - 命中 / 被击 / 暴击时画面抖动
===================================================== */
const ScreenShakeModule = {
  init(){
    let shakeFrame = 0;
    let intensity = 0;

    EventBus.on("animation.shake", ({ power=2, frames=10 })=>{
      intensity = power;
      shakeFrame = frames;
    });

    EventBus.on("game.tick", ()=>{
      if(shakeFrame <= 0) return;

      const canvas = StateStore.get("pixel.canvas");
      const offsetX = (Math.random()*2-1) * intensity;
      const offsetY = (Math.random()*2-1) * intensity;

      canvas.style.transform =
        `translate(${offsetX}px, ${offsetY}px)`;

      shakeFrame--;

      if(shakeFrame === 0){
        canvas.style.transform = "translate(0,0)";
      }
    });

    console.log("[13.0.6 ScreenShakeModule] ready");
  }
};
ModuleRegistry.register(ScreenShakeModule);
/* =====================================================
   [13.0.7] FlashWhiteModule
   - 命中瞬间白闪
===================================================== */
const FlashWhiteModule = {
  init(){

    EventBus.on("animation.flash", ()=>{
      const canvas = StateStore.get("pixel.canvas");
      if(!canvas) return;

      canvas.style.filter = "brightness(3)";
      setTimeout(()=>{
        canvas.style.filter = "none";
      }, 50);
    });

    console.log("[13.0.7 FlashWhiteModule] ready");
  }
};
ModuleRegistry.register(FlashWhiteModule);
/* =====================================================
   [13.0.8] DamagePopupModule
   - 普通伤害 / 暴击红字
===================================================== */
const DamagePopupModule = {
  init(){

    const popups = [];

    EventBus.on("combat.damage.final", ({ damage, crit, target })=>{
      popups.push({
        value: damage,
        crit,
        x: target === "player" ? 0 : 20,
        y: 0,
        life: 30
      });

      EventBus.emit("animation.shake", {
        power: crit ? 4 : 2,
        frames: crit ? 15 : 8
      });

      EventBus.emit("animation.flash");
    });

    EventBus.on("game.tick", ()=>{
      const ctx   = StateStore.get("pixel.ctx");
      const scale = StateStore.get("pixel.scale");

      popups.forEach(p=>{
        ctx.fillStyle = p.crit ? "#ff3333" : "#ffffff";
        ctx.font = `${8*scale}px monospace`;
        ctx.fillText(
          p.value,
          p.x * scale,
          p.y * scale
        );

        p.y -= 0.2;
        p.life--;
      });

      for(let i=popups.length-1;i>=0;i--){
        if(popups[i].life<=0) popups.splice(i,1);
      }
    });

    console.log("[13.0.8 DamagePopupModule] ready");
  }
};
ModuleRegistry.register(DamagePopupModule);
/* =====================================================
   [13.0.9] PixelCombatEffectBinder
   - 战斗事件 → 像素反馈
===================================================== */
const PixelCombatEffectBinder = {
  init(){

    EventBus.on("combat.player.attack", ()=>{
      EventBus.emit("animation.shake",{ power:2, frames:6 });
    });

    EventBus.on("combat.enemy.attack", ()=>{
      EventBus.emit("animation.shake",{ power:2, frames:6 });
    });

    EventBus.on("enemy.dead", ()=>{
      EventBus.emit("animation.flash");
      EventBus.emit("animation.shake",{ power:6, frames:20 });
    });

    console.log("[13.0.9 PixelCombatEffectBinder] bound");
  }
};
ModuleRegistry.register(PixelCombatEffectBinder);
/* =====================================================
   [14.0.0] PixelEffectKernelModule
   - 所有像素特效统一入口
===================================================== */
const PixelEffectKernelModule = {
  init(){

    StateStore.set("pixel.effect.queue", []);

    EventBus.on("pixel.effect.play", effect=>{
      const q = StateStore.get("pixel.effect.queue");
      q.push(effect);
      StateStore.set("pixel.effect.queue", q);
    });

    EventBus.on("game.tick", ()=>{
      EventBus.emit("pixel.effect.tick");
    });

    console.log("[14.0.0] PixelEffectKernel ready");
  }
};
ModuleRegistry.register(PixelEffectKernelModule);
/* =====================================================
   [14.0.1] ScreenShakeEffectModule
===================================================== */
const ScreenShakeEffectModule = {
  init(){

    EventBus.on("pixel.effect.tick", ()=>{
      const q = StateStore.get("pixel.effect.queue");
      if(q.length===0) return;

      const effect = q[0];
      if(effect.type!=="shake") return;

      const canvas = StateStore.get("pixel.canvas");
      const strength = effect.strength || 2;

      canvas.style.transform =
        `translate(${(Math.random()-0.5)*strength}px,
                   ${(Math.random()-0.5)*strength}px)`;

      effect.frame--;
      if(effect.frame<=0){
        canvas.style.transform = "";
        q.shift();
      }

      StateStore.set("pixel.effect.queue", q);
    });

    /* 战斗触发 */
    EventBus.on("combat.damage.final", ()=>{
      EventBus.emit("pixel.effect.play",{
        type:"shake",
        frame:6,
        strength:3
      });
    });

    console.log("[14.0.1] ScreenShakeEffect ready");
  }
};
ModuleRegistry.register(ScreenShakeEffectModule);
/* =====================================================
   [14.0.2] ScreenFlashEffectModule
===================================================== */
const ScreenFlashEffectModule = {
  init(){

    EventBus.on("pixel.effect.tick", ()=>{
      const q = StateStore.get("pixel.effect.queue");
      if(q.length===0) return;

      const effect = q[0];
      if(effect.type!=="flash") return;

      const ctx = StateStore.get("pixel.ctx");
      const canvas = StateStore.get("pixel.canvas");

      ctx.save();
      ctx.fillStyle = `rgba(255,255,255,${effect.alpha})`;
      ctx.fillRect(0,0,canvas.width,canvas.height);
      ctx.restore();

      effect.alpha -= 0.15;
      if(effect.alpha<=0){
        q.shift();
      }

      StateStore.set("pixel.effect.queue", q);
    });

    EventBus.on("combat.damage.final", ({crit})=>{
      if(!crit) return;

      EventBus.emit("pixel.effect.play",{
        type:"flash",
        alpha:0.6
      });
    });

    console.log("[14.0.2] ScreenFlashEffect ready");
  }
};
ModuleRegistry.register(ScreenFlashEffectModule);
/* =====================================================
   [14.0.3] DamagePopupEffectModule
===================================================== */
const DamagePopupEffectModule = {
  init(){

    StateStore.set("pixel.damage.popups", []);

    EventBus.on("combat.damage.final", ({damage,crit,target})=>{
      const pops = StateStore.get("pixel.damage.popups");

      pops.push({
        value: damage,
        crit,
        x: target==="player"?4:20,
        y: 6,
        frame: 20
      });

      StateStore.set("pixel.damage.popups", pops);
    });

    EventBus.on("game.tick", ()=>{
      const ctx   = StateStore.get("pixel.ctx");
      const scale = StateStore.get("pixel.scale");
      const pops  = StateStore.get("pixel.damage.popups");

      for(let i=pops.length-1;i>=0;i--){
        const p = pops[i];

        ctx.save();
        ctx.fillStyle = p.crit ? "#ff3333" : "#ffffff";
        ctx.font = `${6*scale}px monospace`;
        ctx.fillText(
          p.value,
          p.x*scale,
          p.y*scale
        );
        ctx.restore();

        p.y -= 0.3;
        p.frame--;

        if(p.frame<=0) pops.splice(i,1);
      }

      StateStore.set("pixel.damage.popups", pops);
    });

    console.log("[14.0.3] DamagePopupEffect ready");
  }
};
ModuleRegistry.register(DamagePopupEffectModule);

<!-- =====================================================
     [15.0.0] EngineRunModule
     - 最终执行模块
     - 职责：
       - 启动 Kernel
       - 不参与任何系统逻辑
===================================================== -->
<script>
const EngineRunModule = {
  init(){
    // 延迟一帧，确保所有模块已 register
    requestAnimationFrame(()=>{
      ENGINE_BOOT();
      console.log("[15.0.0 EngineRunModule] ENGINE_BOOT called");
    });
  }
};

ModuleRegistry.register(EngineRunModule);
</script>
/* =====================================================
   [15.0.1] PlayerDerivedInitModule
   - 初始化玩家派生数值
===================================================== */
const PlayerDerivedInitModule = {
  init(){

    EventBus.on("game.start", ()=>{

      const player = StateStore.get("player.entity");
      if(!player) return;

      // 初始属性点（全 0）
      const attr = { STR:0, DEX:0, VIT:0, AGI:0, CTR:0 };

      const derived = PlayerStatCalculator.calc(attr);

      StateStore.set("player.derived", derived);

      // 初始化 HP
      EventBus.emit("player.runtime.init", {
        hp: derived.HP
      });

      EventBus.emit("player.stat.change");

      console.log("[15.0.1] player derived initialized", derived);
    });

  }
};
ModuleRegistry.register(PlayerDerivedInitModule);
/* =====================================================
   [15.0.2] EnemyDerivedBindModule
===================================================== */
const EnemyDerivedBindModule = {
  init(){

    EventBus.on("enemy.spawn.ready", (enemies)=>{

      enemies.forEach(e=>{
        const derived = EnemyStatCalculator.calc(e.type);
        e.derived = derived;
        e.runtime.hp = derived.HP;
        e.runtime.maxHP = derived.HP;
      });

      StateStore.set("enemy.active.list", enemies);

      console.log("[15.0.2] enemy derived bound");
    });

  }
};
ModuleRegistry.register(EnemyDerivedBindModule);

</script>
</body>
</html>
