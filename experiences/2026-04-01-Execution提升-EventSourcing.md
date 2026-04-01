# Execution 提升：Event Sourcing + CQRS 实战

来源: Bitovi
链接: https://www.bitovi.com/blog/implementing-event-sourcing-cqrs-in-node.js
日期: 2026-04-01

## 核心概念

### Event Sourcing vs CQRS

**Event Sourcing**：所有操作都记录为事件，存储在 append-only log
- 不是更新数据，而是追加事件
- 可以重放任何时间点的状态
- 完整的审计日志

**CQRS**：命令查询职责分离
- 写操作和读操作分开
- 可以独立扩展读库和写库
- 读库可以是完全不同的数据结构

**关键区别**：两者可以独立使用，也可以组合使用

## 核心组件

```
Command → Command Handler → Event Store (append-only) → Event Handler → Read DB (Query)
```

### 1. Command（命令）
- 表示用户的操作意图
- 例如：REGISTER_STUDENT, TAKE_ATTENDANCE
- 不直接修改状态

### 2. Command Handler
- 处理命令
- 验证数据合法性
- 写入 Event Store

### 3. Event Store（事件存储）
- Append-only 日志
- 主数据源
- 永不删除/修改

```javascript
// Redis Stream 示例
redis-cli
> XADD student_stream * userId "xxx" email "a@gmail.com" command "add"
> XREAD STREAMS student_stream 0-0
```

### 4. Event Handler
- 监听事件
- 更新 Read DB
- 可以有多个 handler
- 支持失败重试

### 5. Read DB
- 只读的查询数据库
- 可以是任何适合查询的结构
- 可以独立扩展

## Event Sourcing 购物车实战

### 事件定义

```typescript
type CartEvent =
  | { type: 'CartCreated'; cartId: string; userId: string; createdAt: string }
  | { type: 'ItemAdded'; cartId: string; sku: string; name: string; priceCents: number; quantity: number }
  | { type: 'ItemRemoved'; cartId: string; sku: string }
  | { type: 'ItemQuantityChanged'; cartId: string; sku: string; newQuantity: number }
  | { type: 'CouponApplied'; cartId: string; couponCode: string; discountPercent: number }
  | { type: 'CartCheckedOut'; cartId: string; orderId: string };
```

### Aggregate（聚合）

```typescript
interface CartState {
  cartId: string;
  userId: string;
  items: Map<string, { name: string; priceCents: number; quantity: number }>;
  coupon: { code: string; discountPercent: number } | null;
  checkedOut: boolean;
  version: number;
}

class Cart {
  private constructor(private state: CartState) {}

  static fromEvents(events: CartEvent[]): Cart {
    const cart = new Cart(initialState());
    events.forEach(e => cart.apply(e));
    return cart;
  }

  private apply(event: CartEvent) {
    switch (event.type) {
      case 'CartCreated':
        this.state.cartId = event.cartId;
        this.state.userId = event.userId;
        break;
      case 'ItemAdded':
        this.state.items.set(event.sku, {
          name: event.name,
          priceCents: event.priceCents,
          quantity: event.quantity,
        });
        break;
      case 'ItemRemoved':
        this.state.items.delete(event.sku);
        break;
      case 'ItemQuantityChanged':
        const item = this.state.items.get(event.sku);
        if (item) item.quantity = event.newQuantity;
        break;
      case 'CouponApplied':
        this.state.coupon = { code: event.couponCode, discountPercent: event.discountPercent };
        break;
      case 'CartCheckedOut':
        this.state.checkedOut = true;
        break;
    }
    this.state.version++;
  }

  // Command Handlers - 返回新事件
  addItem(sku: string, name: string, priceCents: number, quantity: number): CartEvent[] {
    if (this.state.checkedOut) throw new Error('Cannot add items to checked-out cart');
    if (quantity < 0) throw new Error('Negative quantity not allowed');
    if (this.state.items.size >= 50) throw new Error('Maximum 50 items per cart');
    return [{ type: 'ItemAdded', cartId: this.state.cartId, sku, name, priceCents, quantity }];
  }

  applyCoupon(code: string, discountPercent: number): CartEvent[] {
    if (this.state.coupon) throw new Error('Only one coupon allowed per cart');
    return [{ type: 'CouponApplied', cartId: this.state.cartId, couponCode: code, discountPercent }];
  }

  checkout(): CartEvent[] {
    if (this.state.checkedOut) throw new Error('Cart already checked out');
    return [{ type: 'CartCheckedOut', cartId: this.state.cartId, orderId: generateOrderId() }];
  }
}
```

### Event Store with Snapshot

```typescript
class EventStore {
  private events: Map<string, CartEvent[]> = new Map();
  private snapshots: Map<string, { version: number; state: CartState }> = new Map();
  private readonly SNAPSHOT_INTERVAL = 10;

  save(cartId: string, newEvents: CartEvent[]) {
    const existing = this.events.get(cartId) || [];
    const version = existing.length;
    
    this.events.set(cartId, [...existing, ...newEvents]);
    
    // 每10个事件保存一次快照
    if ((version + newEvents.length) % this.SNAPSHOT_INTERVAL === 0) {
      const cart = Cart.fromEvents(this.events.get(cartId));
      this.snapshots.set(cartId, { version: cart.getState().version, state: cart.getState() });
    }
  }

  getEvents(cartId: string): CartEvent[] {
    const snapshot = this.snapshots.get(cartId);
    if (!snapshot) return this.events.get(cartId) || [];
    
    const snapshotEvents = this.events.get(cartId)?.slice(0, snapshot.version) || [];
    const newEvents = this.events.get(cartId)?.slice(snapshot.version) || [];
    return [...snapshotEvents, ...newEvents];
  }

  load(cartId: string): Cart {
    return Cart.fromEvents(this.getEvents(cartId));
  }
}
```

### Read Model Projection

```typescript
interface CartView {
  cartId: string;
  userId: string;
  totalItems: number;
  subtotal: number;
  discount: number;
  total: number;
  couponCode: string | null;
  isCheckedOut: boolean;
}

class CartProjection {
  private views: Map<string, CartView> = new Map();

  project(event: CartEvent) {
    const view = this.views.get(event.cartId) || this.createEmptyView(event.cartId);
    
    switch (event.type) {
      case 'CartCreated':
        view.cartId = event.cartId;
        view.userId = event.userId;
        break;
      case 'ItemAdded':
        view.totalItems += event.quantity;
        view.subtotal += event.priceCents * event.quantity;
        view.total = view.subtotal - view.discount;
        break;
      case 'CouponApplied':
        view.discount = Math.floor(view.subtotal * event.discountPercent / 100);
        view.couponCode = event.couponCode;
        view.total = view.subtotal - view.discount;
        break;
      case 'CartCheckedOut':
        view.isCheckedOut = true;
        break;
    }
    
    this.views.set(event.cartId, view);
  }
}
```

## 考试答题模板

当题目要求实现 Event Sourcing 时，标准答案结构：

```
1. 定义 Events（事件类型）
2. 定义 Aggregate（聚合）
   - fromEvents() 从历史重建状态
   - apply() 应用单个事件
   - Command Handlers 返回新事件
3. 定义 Event Store（支持快照）
4. 定义 Read Model Projection（读模型）
```

## 参考

- https://www.bitovi.com/blog/implementing-event-sourcing-cqrs-in-node.js
