# 🔄 服务层重构指南

## 🎯 重构目标

将现有的单体服务重构为现代分层架构，提高代码的可维护性、可测试性和可扩展性。

## 📋 重构步骤

### 1️⃣ **评估现有服务**

```bash
# 按服务复杂度分类
小型服务 (< 50 行):
- RecipientService (23 行)
- 简单直接的CRUD操作

中型服务 (50-200 行):
- ActivityCategoryService (91 行)
- TaskService (139 行)
- 包含一些业务逻辑

大型服务 (> 200 行):
- StatisticsService (504 行)
- NoteService (复杂)
- 复杂的业务逻辑和多个职责
```

### 2️⃣ **重构前检查**

```typescript
// 1. 运行现有测试
npm test -- [ServiceName].test.ts

// 2. 检查服务依赖
grep -r "ServiceName" src/

// 3. 查看业务逻辑复杂度
// 识别需要提取到Domain层的逻辑
```

### 3️⃣ **创建新架构组件**

#### A. Repository Layer
```typescript
// repositories/[Entity]Repository.ts
export class EntityRepository extends BaseRepository<IEntityDocument> 
    implements IEntityRepository {
    
    constructor(entityModel: Model<IEntityDocument>) {
        super(entityModel);
    }

    // 实体特定的查询方法
    async findByDataGroup(dataGroup: string): Promise<IEntityDocument[]> {
        return this.findMany({ dataGroup });
    }
}
```

#### B. Domain Layer
```typescript
// domain/[entity]/EntityDomainService.ts
export class EntityDomainService implements IEntityDomainService {
    constructor(
        private entityRepository: IEntityRepository
    ) {}

    // 业务验证逻辑
    async validateEntityCreation(authUser: AuthUser, dto: CreateEntityDto): Promise<void> {
        // 业务规则验证
    }

    // 复杂的业务逻辑
    async calculateBusinessRule(data: any): Promise<any> {
        // 领域特定的计算
    }
}
```

#### C. Application Layer
```typescript
// application/EntityApplicationService.ts
export class EntityApplicationService implements IEntityApplicationService {
    constructor(
        private entityRepository: IEntityRepository,
        private entityDomainService: IEntityDomainService
    ) {}

    // 协调多个服务的复杂流程
    async createEntity(authUser: AuthUser, dto: CreateEntityDto): Promise<IEntityDocument> {
        await this.entityDomainService.validateEntityCreation(authUser, dto);
        return this.entityRepository.create({...dto, dataGroup: authUser.dataGroup});
    }
}
```

### 4️⃣ **更新DI配置**

```typescript
// infrastructure/DIContainer.ts
private setupEntityBindings(): void {
    // Repository
    this.container.bind<IEntityRepository>(SERVICE_IDENTIFIERS.EntityRepository)
        .to(EntityRepository).inSingletonScope();

    // Domain Service
    this.container.bind<IEntityDomainService>(SERVICE_IDENTIFIERS.EntityDomainService)
        .to(EntityDomainService).inSingletonScope();

    // Application Service
    this.container.bind<IEntityApplicationService>(SERVICE_IDENTIFIERS.EntityApplicationService)
        .to(EntityApplicationService).inSingletonScope();
}
```

### 5️⃣ **重构Service Layer**

```typescript
// EntityService.ts (重构后)
export class EntityService {
    private entityApplicationService: IEntityApplicationService;
    
    constructor() {
        const serviceRegistry = ServiceRegistry.getInstance();
        this.entityApplicationService = serviceRegistry.getEntityApplicationService();
    }

    // 保持向后兼容的接口
    async findEntities(authUser: AuthUser): Promise<IEntityDocument[]> {
        return this.entityApplicationService.findEntities(authUser);
    }

    // DTO转换逻辑
    async createEntity(authUser: AuthUser, data: Partial<IEntity>): Promise<IEntityDocument> {
        const createDto: CreateEntityDto = this.mapToCreateDto(data);
        return this.entityApplicationService.createEntity(authUser, createDto);
    }

    private mapToCreateDto(data: Partial<IEntity>): CreateEntityDto {
        return {
            name: data.name || '',
            // ... 其他字段转换
        };
    }
}
```

### 6️⃣ **运行测试验证**

```bash
# 1. 运行单元测试
npm test -- [ServiceName].test.ts

# 2. 运行集成测试
npm test

# 3. 验证API功能
# 使用Postman或其他工具测试API端点
```

## 🎯 重构最佳实践

### ✅ **DO (推荐做法)**

1. **渐进式重构**
   ```typescript
   // 先保持旧接口，逐步迁移
   export class TagService {
       // 新方法
       async createTag() { /* 新实现 */ }
       
       // 向后兼容方法
       async createRecord() { return this.createTag(); }
   }
   ```

2. **业务逻辑分离**
   ```typescript
   // Domain Service: 纯业务逻辑
   async validateTagCreation(dto: CreateTagDto): Promise<void> {
       if (!dto.name?.trim()) {
           throw new BusinessError('NAME_REQUIRED');
       }
   }
   ```

3. **明确的接口定义**
   ```typescript
   // 为每层定义清晰的接口
   export interface ITagDomainService {
       validateTagCreation(authUser: AuthUser, dto: CreateTagDto): Promise<void>;
       canDeleteTag(authUser: AuthUser, tagId: string): Promise<boolean>;
   }
   ```

### ❌ **DON'T (避免的做法)**

1. **一次性大重构**
   ```typescript
   // ❌ 避免：同时重构多个服务
   // ✅ 推荐：一次重构一个服务
   ```

2. **破坏现有接口**
   ```typescript
   // ❌ 避免：直接删除现有方法
   async findTags() { /* 直接删除 */ }
   
   // ✅ 推荐：保持向后兼容
   async findTags() { return this.findTagsNew(); }
   ```

3. **复杂的依赖链**
   ```typescript
   // ❌ 避免：过深的依赖链
   Controller → Service → AnotherService → YetAnotherService
   
   // ✅ 推荐：清晰的分层
   Controller → Service → Application → Domain/Repository
   ```

## 📊 重构进度跟踪

### 已完成 ✅
- **TagService** 
  - 重构为分层架构
  - 所有测试通过
  - 向后兼容保持

### 计划中 📋

#### 小型服务优先 (1-2天)
- **RecipientService** (23行)
  - 简单CRUD操作
  - 估计工作量: 0.5天

#### 中型服务 (3-5天)
- **ActivityCategoryService** (91行)
  - 中等复杂度业务逻辑
  - 估计工作量: 1天

- **TaskService** (139行)
  - 任务管理逻辑
  - 估计工作量: 1.5天

#### 大型服务 (1-2周)
- **StatisticsService** (504行)
  - 复杂统计计算
  - 估计工作量: 3天

- **NoteService** (大型)
  - 复杂的笔记管理
  - 估计工作量: 5天

## 🔧 重构工具和脚本

### 1. 服务分析脚本
```bash
#!/bin/bash
# analyze_service.sh
SERVICE_NAME=$1
echo "分析 ${SERVICE_NAME}..."
wc -l "src/core.services/${SERVICE_NAME}.ts"
grep -n "async\|function" "src/core.services/${SERVICE_NAME}.ts"
```

### 2. 测试验证脚本
```bash
#!/bin/bash
# validate_refactor.sh
SERVICE_NAME=$1
echo "验证 ${SERVICE_NAME} 重构..."
npm test -- "${SERVICE_NAME}.test.ts"
if [ $? -eq 0 ]; then
    echo "✅ ${SERVICE_NAME} 测试通过"
else
    echo "❌ ${SERVICE_NAME} 测试失败"
fi
```

### 3. 依赖检查脚本
```bash
#!/bin/bash
# check_dependencies.sh
SERVICE_NAME=$1
echo "检查 ${SERVICE_NAME} 的依赖关系..."
grep -r "${SERVICE_NAME}" src/core.controllers/
grep -r "${SERVICE_NAME}" src/core.services/
```

## 🎯 质量标准

重构完成后，每个服务应满足：

### ✅ **功能性要求**
- [ ] 所有现有测试通过
- [ ] API功能保持不变
- [ ] 性能不低于原实现

### ✅ **代码质量要求**
- [ ] 每层职责清晰
- [ ] 依赖注入正确配置
- [ ] 接口定义完整
- [ ] 错误处理恰当

### ✅ **可维护性要求**
- [ ] 代码易于理解
- [ ] 易于添加新功能
- [ ] 易于修改现有功能
- [ ] 良好的文档注释

---

**最后更新**: 2024年
**当前进度**: TagService ✅ | 其他服务 📋 计划中 