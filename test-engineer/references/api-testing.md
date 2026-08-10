# 后端 API 测试(接口测试 / NestJS / 压测)

## 手工接口测试(curl / httpie)

curl 是最通用的重放与验证工具,测试常用形态:

```bash
# GET + 查看响应头和耗时
curl -s -i "https://api.example.com/users?page=1"
curl -s -o /dev/null -w "status:%{http_code} total:%{time_total}s ttfb:%{time_starttransfer}s\n" URL

# POST JSON
curl -s -X POST https://api.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"username":"user1","password":"pass1"}'

# 带认证
curl -s -H "Authorization: Bearer $TOKEN" https://api.example.com/me

# 文件上传
curl -s -F "file=@photo.jpg" -F "type=avatar" https://api.example.com/upload

# 响应 JSON 处理(配合 jq)
curl -s URL | jq '.data.items | length'
curl -s URL | jq -e '.code == 0' > /dev/null && echo PASS || echo FAIL
```

从浏览器 DevTools "Copy as cURL" 拿到的命令,是复现前端请求的最快方式;把它逐项精简(删 header)可以定位"哪个 header/参数导致行为不同"。

## 接口测试的检查清单

对一个接口做测试,系统性地过这几类,而不是只测 happy path:

1. **正向**:合法输入 → 正确输出(响应结构、字段类型、业务值)
2. **参数校验**:缺参数、参数类型错、超长字符串、空字符串 vs null、特殊字符/emoji、SQL 注入形态的字符串(`' OR 1=1--`,预期应被当普通字符串处理)
3. **认证与权限**:无 token、过期 token、他人的 token 访问自己的资源(越权,重点!)、低权限角色调高权限接口
4. **边界**:分页 page=0/-1/超大值、金额 0/负数/超大/小数精度、列表空数据
5. **幂等与重复**:同一请求连发两次(重复下单、重复支付回调)结果是否正确
6. **错误码约定**:失败时 HTTP 状态码和业务 code 是否符合文档,错误信息是否泄露内部实现(堆栈、SQL)
7. **并发**(必要时):同一资源并发修改是否有竞态(如库存扣减)

## NestJS 测试(Jest + supertest)

用户在学 NestJS,写例子时注意解释 Nest 特有的测试模块机制。

### 单元测试(隔离 service,mock 依赖)

```ts
// users.service.spec.ts
import { Test } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { User } from './user.entity';

describe('UsersService', () => {
  let service: UsersService;
  const mockRepo = { findOneBy: jest.fn(), save: jest.fn() };

  beforeEach(async () => {
    // Test.createTestingModule 是 Nest 的测试容器:
    // 像真实 AppModule 一样做依赖注入,但可以用 mock 替换任意 provider
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: getRepositoryToken(User), useValue: mockRepo },
      ],
    }).compile();
    service = module.get(UsersService);
    jest.clearAllMocks();
  });

  it('用户不存在时抛 NotFoundException', async () => {
    mockRepo.findOneBy.mockResolvedValue(null);
    await expect(service.findOne(999)).rejects.toThrow('User not found');
  });
});
```

### E2E 测试(起完整应用,supertest 发真实 HTTP)

```ts
// test/users.e2e-spec.ts
import { Test } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Users (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true })); // 和 main.ts 保持一致,否则测的不是真实行为
    await app.init();
  });
  afterAll(async () => { await app.close(); });

  it('POST /users 缺少 email 时返回 400', () =>
    request(app.getHttpServer())
      .post('/users')
      .send({ name: 'u1' })
      .expect(400)
      .expect(res => {
        expect(res.body.message).toEqual(expect.arrayContaining([expect.stringContaining('email')]));
      }));
});
```

要点:

- E2E 的数据库用独立测试库或 testcontainers,每个用例前清理数据,禁止连开发库
- `main.ts` 里的全局 pipe/filter/interceptor 在测试里要同样装上,否则校验行为不一致
- 外部依赖(第三方支付、短信)在 E2E 层用 `overrideProvider()` 换成 fake
- 运行:`npm run test`(单测)、`npm run test:e2e`、`npm run test:cov`(覆盖率)

## 压力测试(k6 首选)

先定目标再压:目标 QPS、可接受的 P95/P99 延迟、错误率上限。没有目标的压测得不出结论。

```bash
brew install k6   # 或 docker run -i grafana/k6
k6 run load-test.js
```

```js
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // 逐级加压,不要一步到位
    { duration: '1m',  target: 50 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // P95 < 500ms,超了 k6 直接判失败
    http_req_failed: ['rate<0.01'],    // 错误率 < 1%
  },
};

export default function () {
  const res = http.post('https://api.example.com/login',
    JSON.stringify({ username: `user${__VU}`, password: 'pass1' }),
    { headers: { 'Content-Type': 'application/json' } });
  check(res, {
    'status 200': r => r.status === 200,
    'has token': r => !!r.json('token'),
  });
  sleep(1);
}
```

报告必含:并发数/QPS、P50/P95/P99 延迟、错误率、以及压测期间服务端资源(CPU/内存/慢查询)——只有客户端数据没有服务端数据的压测报告是半成品。快速小工具:`ab -n 1000 -c 50 URL`(apache bench)适合一次性冒烟,正式压测用 k6。

注意:压测只能打自己有权限的环境,压生产要走审批和限流预案。
