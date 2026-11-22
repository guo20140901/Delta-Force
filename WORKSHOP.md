# 三角洲 2D 搜打撤 · VibeHub 创意工坊 Mod 开发指南

本文件是《三角洲 2D 搜打撤》(Delta 2D) 的工坊作者—创作者约定。平台不会扫描或限制 Mod 代码，
只会读取 `WORKSHOP_CAPABILITIES.json` 决定 UI 上展示哪些能力。

本游戏是单机模式，没有联机匹配，且联机策略为 `mods-disabled`：所有 Mod 只影响本地表现。

- 游戏版本：v3.2.0（工坊层 v1）
- Mod API 版本：`window.VibeHubGame`（`workshop: "v1"`）
- 入口：`const api = window.VibeHubGame.modApi;`

---

## 1. 玩家可以制作哪些 Mod

| 类别 | 示例 | 用到的主要 API |
|---|---|---|
| 新武器 / 改枪 | 「M4A1 黄金版：伤害 +15%」「一把 SCAR-H」 | `api.weapons.register(key, def)` |
| NPC 强化 | 「敌方血量 ×1.5、反应更快」「首领必带 SR-25」 | `api.npcs.registerTemplate(tpl)` |
| 玩家增益 | 「血量上限 150」「击杀回血 10」 | `api.player.hookDamage` + `api.events.on('kill')` |
| 子弹规则 | 「穿甲弹：命中后 30% 伤害穿透到下一个目标」 | `api.player.hookFireBullet` |
| 事件驱动 | 「每次击杀 +¥50」「撤离时提示」 | `api.events.on(event, fn)` |
| 物资调整 | 「所有容器必出 1 个血包」 | `api.loot.hookRoll(fn)` |
| 地图装饰 | 「在撤离点附近加一圈路灯」 | `api.maps.decorate(fn)` |

多 Mod 叠加：`weapons.register` 同 key 后注册覆盖先注册；`registerTemplate` 同 id 后注册覆盖先注册；
模板之间按「相乘」叠加；钩子按注册顺序串联，任一返回值即改写。

---

## 2. 加载阶段

| 阶段 | 适合做的事 | 不适合做的事 |
|---|---|---|
| `before-start` | 注册武器、NPC 模板、地图装饰 | 订阅运行时事件、改写运行中的敌人 |
| `after-start` | 订阅事件、注册伤害/子弹/搜索钩子、查询状态 | 注册新武器（交易行已渲染） |

执行顺序：

```
Loader → before-start Mod（并行，按 dependencyIds 拓扑序）
→ 游戏暴露 window.VibeHubGame.modApi
→ 玩家点「开始行动」→ 对局开始 → 触发 round:start
→ 游戏调用 VibeHubWorkshop.markGameReady()
→ Loader 拉取 after-start Mod → api.phase.ready 变为 true，onReady 回调触发
```

- before-start Mod 可安全调用：`weapons.*`、`npcs.registerTemplate`、`maps.decorate`、`events.on`（仅注册）。
- before-start 阶段不要调用 `player.heal` / `hookDamage` / `hookFireBullet`（此时还没有对局）。
- after-start Mod 请先 `await` 就绪：`await new Promise(r => api.phase.ready ? r() : api.phase.onReady(r));`

---

## 3. API 一览

| API | 用途 | 阶段 |
|---|---|---|
| `api.weapons.register(key, def)` | 注册/覆盖武器到 `WEAPONS`，并刷新交易行 | before-start |
| `api.weapons.get(key)` | 读单个武器快照，不存在返回 `null` | 任意 |
| `api.weapons.listAll()` | 所有武器 key | 任意 |
| `api.weapons.listModded()` | 本 Mod 注册过的 key | 任意 |
| `api.npcs.registerTemplate(tpl)` | 注册敌方模板，注入后续生成的敌人 | before-start |
| `api.npcs.applyTo(enemy)` | 对手上敌人实例立即应用全部模板 | after-start |
| `api.npcs.snapshot()` | 存活敌人快照数组（只读） | 任意 |
| `api.npcs.spawnSquad(spec)` | 按编制空降一支小队，自动避让玩家与其他单位 | after-start |
| `api.npcs.squads()` | 存活小队概览（人数、坐标） | after-start |
| `api.maps.decorate(fn)` | 注册地图装饰回调 | before-start |
| `api.player.snapshot()` | 玩家快照，未开局返回 `null` | after-start |
| `api.player.heal(amount)` | 加血，不超过上限，返回实际恢复量 | after-start |
| `api.player.hookDamage(fn)` | 玩家受击前回调 | after-start |
| `api.player.hookFireBullet(fn)` | 开火前回调 | after-start |
| `api.loot.hookRoll(fn)` | 容器搜索产物回调 | after-start |
| `api.events.on / off / emit` | 事件订阅、退订、广播 | 任意 |
| `api.refs.WEAPONS / ITEMS / ENEMY_TYPES / EXTRACTS` | 只读字典引用 | 任意 |
| `api.phase.ready / api.phase.onReady(cb)` | 运行时钩子就绪状态 | 任意 |

### 3.0 阵营与小队（v3.2 新增）

> 本节全部 API 归在 **`npcs.apply-to`** 能力之下——平台的作者能力目录里没有单独的「生成小队」条目，
> Mod 清单声明 `npcs.apply-to` 即可使用 `spawnSquad` / `squads()`，不要另造新的能力 id，否则发布时会被拒。

游戏 v3.2 起内置四阵营敌友矩阵，敌对关系对称。**注意 `gti` 内部互相敌对**（还原 GTI 内斗：干员之间、干员与玩家都会交火），
`sara` 与 `hafk` 内部友好：

| 阵营 id | 归属 |
|---|---|
| `player` | 玩家（归入 GTI 一方，与干员敌对） |
| `sara` | 阿萨拉（默认野怪、盾兵、机枪兵、首领赛伊德） |
| `gti` | 敌方干员（GTI） |
| `hafk` | 哈夫克（默认不存在，由 Mod 生成） |

子弹按**射手阵营**结算：哈夫克打阿萨拉、阿萨拉还击哈夫克都能成立，`kill` 事件也会照常触发。

`api.npcs.spawnSquad(spec)` 生成一支小队：

```js
api.npcs.spawnSquad({
  name: '哈夫克·第 1 小队',
  faction: 'hafk',          // 默认 hafk
  minPlayerDist: 1100,      // 出生点距玩家最小距离，默认 1000
  minEnemyDist: 300,        // 距其它单位最小距离，默认 320
  area: {x:1600, y:2000, w:3000, h:2400},  // 可选，限定落区
  // x: 4800, y: 3000,      // 可选，直接指定坐标（跳过搜索）
  members: [                // 每个成员一份内联 def
    { role: 'rifleman', name: '哈夫克小兵', hp: 70, damage: 9, range: 420 },
    { role: 'shield', name: '哈夫克盾兵', hp: 130, hasShield: true, radius: 14 }
  ]
});
// → { id, name, faction, count, x, y }；找不到落点返回 null
```

- `members[0]` 是队长，其余成员自动跟随（距队长 > 190 时靠拢，交战中不跟随）。
- `role` 决定地图符号与掉落表：`shield` ▣ / `mg` ▮ / `sniper` ◎ / 其它 `·`；
  掉落走游戏内已配好的 `ENEMY_DROPS['hafk_' + role]`。
- 出生点搜索最多采样 90 次，要求不压墙、距玩家 ≥ `minPlayerDist`、距所有存活单位 ≥ `minEnemyDist`。
- 每支小队触发 `squad:spawn` 事件（`{id, name, faction, count, x, y, members}`）。
- `api.npcs.squads()` → `[{id, name, faction, alive, total, x, y}]`，只读。
- 对局里野怪生成完毕后会广播 **`round:spawned`**。`round:start` 时场上还没有敌人，
  不要在 `round:start` 里刷小队，监听 `round:spawned` 才能避开它们。

### 3.1 武器定义 `def`

```js
{
  name: 'SCAR-H',        // 必填，展示名
  icon: '▬',             // 可选，单字符图标
  price: 12000,          // 可选，哈夫币价格，0 为免费
  damage: 1.45,          // 倍率，基数为干员伤害
  fireRate: 1.05,        // 倍率，越大越快
  range: 1.15,           // 倍率
  mag: 20,               // 弹匣容量，最小 1
  ammoType: 'heavy',     // light / medium / heavy / shell，非法值回落 medium
  desc: '7.62mm 战斗步枪'
}
```

非法数值会回落到默认值，不会污染 `WEAPONS` 其他条目。

### 3.2 NPC 模板 `tpl`

```js
{
  id: 'mymod_hard',   // 必填，建议带 mod 前缀
  hpMul: 1.5,         // 生命倍率
  dmgMul: 1.2,        // 伤害倍率
  speedMul: 1.1,      // 移动速度倍率
  viewMul: 1.2,       // 视野半径倍率
  fireRateMul: 0.9,   // 射击冷却倍率（越小越快）
  reactMul: 0.8,      // 反应时间倍率（越小反应越快）
  radiusMul: 1.0,     // 体型倍率
  weapon: 'sr25'      // 强制指定武器 key（仅对已有该字段的敌人有意义）
}
```

模板只影响注册之后生成的敌人。想让首领必带 SR-25，可写 `{ id:'boss_sr25', weapon:'sr25' }`。

### 3.3 钩子签名

```js
// 伤害钩子：返回数字 = 改写；返回 undefined = 透传
api.player.hookDamage((amount, attacker, target, info) => {
  if (amount >= 60) return amount * 0.7;
});

// 子弹钩子：返回对象 = 合并替换；返回 undefined = 透传
api.player.hookFireBullet((def, shooter, origin, dir) => {
  return { damage: def.damage * 1.3 };
});

// 搜索产物钩子：返回数组 = 替换；返回 undefined = 透传
api.loot.hookRoll((items, container) => {
  const med = api.refs.ITEMS.med238;
  if (med && !items.some(it => it.id === 'med238')) items.push({ ...med });
  return items;
});
```

### 3.4 地图装饰

`decorate` 回调收到装饰数组，直接 push 即可；返回数组则替换整个列表。

```js
api.maps.decorate((list) => {
  list.push({ x: 4000, y: 2900, type: 'lamp', size: 12, alpha: 0.4 });
});
```

支持的 `type`：`grass`、`rock`、`crack`、`debris`、`lamp`（暖光点）、`marker`（金色方块）；
其他 type 会画成一个浅灰小圆点。装饰只影响渲染，不参与碰撞。

### 3.5 事件

| 事件 | payload | 触发时机 |
|---|---|---|
| `round:start` | `{ operator, difficulty }` | 对局开始 |
| `round:end` | `{ reason, success }` | 撤离或阵亡结算 |
| `extract` | `{ extractId, success }` | 撤离成功 |
| `kill` | `{ attacker, victim, isHead }` | 玩家击杀记入击杀播报 |
| `death` | `{ attacker, isHead, target }` | 玩家阵亡 |
| `damage` | `{ amount, attacker, target, info }` | 玩家受击，`amount` 为钩子处理后的最终值 |
| `loot:taken` | `{ item }` | 从容器或战利品包拾取物品 |
| `weapon:registered` | `{ key, def, replaced }` | 任意 Mod 调用 `weapons.register` |
| `npc:registered` | `{ id }` | 任意 Mod 调用 `registerTemplate` |

`api.events.emit(name, payload)` 可广播自定义事件，供其他 Mod 订阅。

---

## 4. 最小可运行 Mod

### 4.1 启动前（`loadPhase: before-start`）

```js
(function () {
  if (!window.VibeHubGame || !window.VibeHubGame.modApi) return;
  const api = window.VibeHubGame.modApi;

  api.weapons.register('mymod_scarh', {
    name: 'SCAR-H', icon: '▬', price: 12000,
    damage: 1.45, fireRate: 1.05, range: 1.15,
    mag: 20, ammoType: 'heavy', desc: '7.62mm 战斗步枪'
  });

  api.npcs.registerTemplate({ id: 'mymod_tougher', hpMul: 1.2, reactMul: 0.9 });

  api.maps.decorate((list) => {
    list.push({ x: 4000, y: 2900, type: 'lamp', size: 12, alpha: 0.4 });
  });
})();
```

### 4.2 启动后（`loadPhase: after-start`）

```js
(async function () {
  if (!window.VibeHubGame || !window.VibeHubGame.modApi) return;
  const api = window.VibeHubGame.modApi;

  await new Promise((r) => (api.phase.ready ? r() : api.phase.onReady(r)));

  api.player.hookDamage((amount) => (amount >= 60 ? amount * 0.7 : undefined));

  api.events.on('kill', ({ victim }) => {
    if (victim) api.player.heal(8);
  });

  api.events.on('round:end', ({ reason, success }) => {
    console.log('[mod] 对局结束', success ? '撤离成功' : '阵亡', reason);
  });
})();
```

---

## 5. 命名、去重与冲突

- 武器 key、模板 id 用小写英文 + 数字 + 下划线，并带 Mod 前缀，例如 `mymod_scarh`。
- `weapons.register` 同 key 后注册覆盖先注册；`registerTemplate` 同 id 后注册覆盖先注册。
- `maps.decorate` 多个回调按注册顺序全部执行。
- 三个钩子按注册顺序串联，任一返回值即改写。
- 不要独占某个 key；不要阻塞主线程（同步循环超过 100ms 会掉帧）。

---

## 6. 明确禁止

| 禁止 | 原因 |
|---|---|
| 直接读写 `localStorage` / `indexedDB` | 与游戏存档 schema 冲突，存档走 `vibe.save` |
| 操作 DOM | HUD 结构可能在任意小版本调整 |
| 覆盖 `window.VibeHubGame` / `window.game` | 会破坏后续 Mod |
| 直接 mutate `WEAPONS` / `ITEMS` / `ENEMY_TYPES` 而不走 register | 注册表才是稳定接触面 |
| 直接修改 `Game.enemies` 数组 | 请用 `npcs.snapshot()` + 自己的状态机 |
| 读写网络层 / WebSocket / fetch | 平台不为 Mod 网络行为背书 |
| 绕过 Loader | 破坏工坊体系 |

能力声明不是安全沙箱。平台不会阻止越权代码，越权带来的后果由 Mod 作者承担。

---

## 7. 版本兼容

工坊层 v1 与游戏 v3.x 绑定。**v3.2 起才支持 `npcs.spawnSquad` / 阵营矩阵 / `round:spawned`**，
在 v3.0 / v3.1 上调用这些能力会拿到 `null` 或 `undefined`。大版本升级可能修改 `WEAPONS` / `ENEMY_TYPES` / `ITEMS` 字段；
建议在 Mod 描述里写明兼容版本范围。若平台检测到能力 ID 已被移除，会在 Mod 列表标红「已不兼容」。
