# 数据结构设计文档

## 术语定义

| 术语 | 含义 |
|------|------|
| **关注者 (Following)** | 该用户关注的人 |
| **追随者 (Follower)** | 关注了该用户的人 |

## API 详细说明

### 已实现的 API

#### 1. 获取当前用户信息

**函数**: `getCurrentUserMidFromAPI()`  
**接口**: `GET https://api.bilibili.com/x/web-interface/nav`  
**用途**: 获取当前登录用户的 uid  
**代码位置**: `src/services/biliApi.ts:79-89`

```typescript
// 返回类型
interface NavResponse {
  code: number;
  message: string;
  data: {
    isLogin: boolean;
    mid: number;      // 用户 uid
    uname: string;    // 用户名
  };
}
```

#### 2. 获取关注列表 API

**函数**: `getFollowingsList(params)`  
**接口**: `GET https://api.bilibili.com/x/relation/followings`  
**用途**: 获取指定用户的关注列表  
**限制**: 用户可设为隐私，分页获取  
**代码位置**: `src/services/biliApi.ts:117-126`

```typescript
// 请求参数
interface GetFansListParams {
  vmid: number;   // 目标用户 uid
  ps?: number;    // 每页数量，默认 20
  pn?: number;    // 页码，默认 1
}

// 返回类型 FansResponse
{
  code: number;
  data: {
    list: FansItem[];  // 关注列表
    total: number;     // 总数
  }
}
```

#### 3. 获取粉丝列表 API

**函数**: `getFansList(params)`  
**接口**: `GET https://api.bilibili.com/x/relation/fans`  
**用途**: 获取指定用户的粉丝（追随者）列表  
**限制**: 用户可设为隐私  
**代码位置**: `src/services/biliApi.ts:106-115`

```typescript
// 请求参数同 getFollowingsList
// 返回类型同 FansResponse
```

#### 4. 获取共同关注 API

**函数**: `getCommonFollowings(vmid, useCache?)`  
**接口**: `GET https://api.bilibili.com/x/relation/followings/followed_upper`  
**用途**: 获取我与目标用户的共同关注  
**特点**: 不受隐私设置影响，但只返回我也关注的人  
**缓存**: 支持本地缓存  
**代码位置**: `src/services/biliApi.ts:131-154`

```typescript
// 请求参数
{ vmid: number }  // 目标用户 uid

// 返回类型 CommonFollowingsResponse
{
  code: number;
  data: {
    desc: string;
    list: FansItem[];  // 共同关注列表
    total: number;
  }
}
```

### 通用返回类型

```typescript
// FansItem - 用户信息项
interface FansItem {
  mid: number;           // 用户 uid
  uname: string;         // 用户名
  face: string;          // 头像 URL
  sign: string;          // 签名
  attribute: number;     // 关系: 0-未关注, 2-已关注, 6-互粉, 128-已拉黑
  mtime: number;         // 关注时间（秒级时间戳）
  official_verify: {     // 认证信息
    type: number;        // -1: 无, 0: UP主, 1: 机构
    desc: string;
  };
  vip: VipInfo;          // 大会员信息
}
```

### API 实现状态

| API | 函数 | 实现状态 | 使用情况 |
|-----|------|---------|---------|
| 获取当前用户信息 | `getCurrentUserMidFromAPI` | 已实现 | 已使用 |
| 获取关注列表 | `getFollowingsList` | 已实现 | 已使用 |
| 获取粉丝列表 | `getFansList` | 已实现 | 仅在 `FansList` 组件中使用，深度探索功能待接入 |
| 获取共同关注 | `getCommonFollowings` | 已实现 | 已使用 |

## API 与数据来源对照表

| API | 函数 | 返回内容 | 限制 | 数据字段 |
|-----|------|---------|------|----------|
| 共同关注 API | `getCommonFollowings` | 我与目标用户的共同关注列表 | 只返回**我也关注的人**，可能有遗漏 | `following` |
| 关注列表 API | `getFollowingsList` | 目标用户的关注者列表 | 用户可设为隐私 | `following`(我) / `deepFollowing`(他人) |
| 追随者列表 API | `getFansList` | 目标用户的追随者列表 | 用户可设为隐私 | `deepFollower` |
| 用户信息 API | `getCurrentUserMidFromAPI` | 当前登录用户信息 | 需要登录 | `myUid` |

## 数据结构

### 统一的用户数据结构

```typescript
interface UserData {
  uid: number;
  uname: string;
  face: string;
  
  /** 
   * 来源: 共同关注 API（对于他人）/ 关注列表 API（对于自己）
   * 对于「我」: 我关注的所有人
   * 对于「他人」: 该用户关注的人中，与我有共同关注的部分
   */
  following: number[];
  
  /**
   * 来源: 关注列表 API
   * 含义: 该用户关注的人（前100个）
   * 深度探索时获取
   */
  deepFollowing: number[];
  
  /**
   * 来源: 追随者列表 API
   * 含义: 关注了该用户的人
   * 深度探索时获取
   */
  deepFollower: number[];
}
```

### 数据存储

```typescript
/** 所有用户数据: uid → UserData */
type UserStore = Map<number, UserData>;

/** 应用状态 */
interface AppState {
  myUid: number;                // 记录谁是我
  users: Map<number, UserData>; // 所有用户统一存储
}
```

## 数据获取流程

```
┌─────────────────────────────────────────────────────────────┐
│                      第一阶段：表层数据                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 获取当前登录用户 uid（通过 cookie/API）                   │
│                              ↓                              │
│  2. 获取「我」的关注列表                                      │
│                              ↓                              │
│  3. 创建「我」的 UserData:                                   │
│     - following = [我关注的所有人]                           │
│     - deepFollowing = []                                    │
│     - deepFollower = []                                     │
│                              ↓                              │
│  4. 遍历我的每个关注，调用「共同关注 API」                     │
│                              ↓                              │
│  5. 为每个关注创建 UserData:                                 │
│     - following = [共同关注返回的 uid 列表]                   │
│     - deepFollowing = []                                    │
│     - deepFollower = []                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              ↓
                      用户点击「深度探索」
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                      第二阶段：深度数据                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  遍历 UserStore 中的每个用户（包括我）:                       │
│                                                             │
│  1. 调用「关注列表 API」                                      │
│     → 成功: deepFollowing = [前100个关注]                    │
│     → 失败(隐私): deepFollowing = []                        │
│                                                             │
│  2. 调用「追随者列表 API」                                    │
│     → 成功: deepFollower = [追随者列表]                      │
│     → 失败(隐私): deepFollower = []                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 「我」与「他人」的字段来源对比

| 字段 | 我 | 他人 |
|------|-----|------|
| `following` | `getFollowingsList(myUid)` 关注列表 API，获取完整关注列表 | `getCommonFollowings(uid)` 共同关注 API，仅返回与我有共同关注的部分，**不完整但一定能获取** |
| `deepFollowing` | **不需要获取**，与 `following` 相同，保持空数组 | `getFollowingsList(uid)` 深度探索时调用，**可能因隐私设置失败返回空数组** |
| `deepFollower` | `getFansList(myUid)` 深度探索时调用，获取谁关注了我 | `getFansList(uid)` 深度探索时调用，**可能因隐私设置失败返回空数组** |

### 详细说明

#### `following` 字段

**对于「我」:**
- 调用 `getFollowingsList({ vmid: myUid, ps: 50, pn: 1~N })`
- 分页获取我的完整关注列表
- 一定能获取成功（自己的数据）

**对于「他人」:**
- 调用 `getCommonFollowings(uid)`
- 返回该用户关注的人中，**同时也被我关注的人**
- 不受对方隐私设置影响，但数据不完整
- 例如：用户 A 关注了 [B, C, D, E]，我关注了 [B, C, F]，则返回 [B, C]

#### `deepFollowing` 字段

**对于「我」:**
- **不需要获取**
- `following` 已经是完整数据，`deepFollowing` 保持空数组即可

**对于「他人」:**
- 调用 `getFollowingsList({ vmid: uid })`
- 尝试获取该用户的关注列表（前 100 个）
- 如果对方设置了隐私，API 返回错误，`deepFollowing = []`
- 如果成功，可补充 `following` 中缺失的数据

#### `deepFollower` 字段

**对于「我」:**
- 调用 `getFansList({ vmid: myUid })`
- 获取关注了我的人（我的粉丝）

**对于「他人」:**
- 调用 `getFansList({ vmid: uid })`
- 尝试获取关注了该用户的人
- 如果对方设置了隐私，API 返回错误，`deepFollower = []`

## 设计说明

### 数据结构不区分身份的好处

1. **统一处理逻辑** - 图渲染、数据分析不需要特殊判断
2. **简化存储** - 一个 Map 存所有人
3. **灵活扩展** - 如果需要知道谁是「我」，只需记录 `myUid` 即可

### 使用示例

```typescript
// 判断是否是我
const isMe = (uid: number) => uid === appState.myUid;

// 获取我的数据
const getMyData = () => appState.users.get(appState.myUid);
```

### 渐进式数据获取策略

1. **保底层** (`following`): 通过共同关注 API 获取，绕过隐私限制
2. **深度层** (`deepFollowing`/`deepFollower`): 尝试获取更完整的数据，失败则降级

这样设计可以在用户隐私设置各异的情况下，尽可能多地收集关系数据用于分析。
