# Supabase 配置指南

## 1. 注册/登录 Supabase

1. 打开 https://supabase.com/
2. 点击 "Start your project"
3. 选择 **Sign up with GitHub**（推荐，避免邮箱验证码）
4. 授权 GitHub 登录

## 2. 创建项目

1. 点击 "New project"
2. 输入项目名称，例如：`kuayangshi-exam`
3. 选择地区：建议选 `Singapore` 或 `Northeast Asia`
4. 等待项目创建完成（约1-2分钟）

## 3. 执行数据库脚本

进入项目后，点击左侧 **SQL Editor** → **New query**，粘贴以下内容并执行：

```sql
-- 启用 UUID 扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 考核记录表
CREATE TABLE IF NOT EXISTS exams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  trainee_name TEXT NOT NULL,
  difficulty TEXT NOT NULL CHECK (difficulty IN ('easy','medium','hard')),
  scenarios JSONB NOT NULL,
  overall_score INTEGER,
  grade TEXT,
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  finished_at TIMESTAMPTZ,
  status TEXT DEFAULT 'in_progress' CHECK (status IN ('in_progress','completed','stopped'))
);

-- 考核消息表
CREATE TABLE IF NOT EXISTS exam_messages (
  id BIGSERIAL PRIMARY KEY,
  exam_id UUID REFERENCES exams(id) ON DELETE CASCADE,
  scenario_id TEXT NOT NULL,
  round_index INTEGER NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('sales','customer')),
  content TEXT NOT NULL,
  score INTEGER,
  feedback TEXT,
  score_details TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 场景分数表
CREATE TABLE IF NOT EXISTS scenario_scores (
  id BIGSERIAL PRIMARY KEY,
  exam_id UUID REFERENCES exams(id) ON DELETE CASCADE,
  scenario_id TEXT NOT NULL,
  total_score INTEGER NOT NULL,
  max_score INTEGER NOT NULL,
  percentage INTEGER NOT NULL,
  UNIQUE(exam_id, scenario_id)
);

-- 考核指标表
CREATE TABLE IF NOT EXISTS exam_metrics (
  id BIGSERIAL PRIMARY KEY,
  exam_id UUID REFERENCES exams(id) ON DELETE CASCADE,
  scenario_id TEXT NOT NULL,
  round_index INTEGER NOT NULL,
  trust_level INTEGER NOT NULL DEFAULT 25,
  intent_level INTEGER NOT NULL DEFAULT 20,
  trust_delta INTEGER NOT NULL DEFAULT 0,
  intent_delta INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 应用配置表（LLM 配置等）
CREATE TABLE IF NOT EXISTS app_config (
  key TEXT PRIMARY KEY,
  value JSONB NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 用户角色扩展表
CREATE TABLE IF NOT EXISTS app_users (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'examinee' CHECK (role IN ('admin','examinee')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ========== RLS（行级安全策略）==========
ALTER TABLE exams ENABLE ROW LEVEL SECURITY;
ALTER TABLE exam_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE scenario_scores ENABLE ROW LEVEL SECURITY;
ALTER TABLE exam_metrics ENABLE ROW LEVEL SECURITY;
ALTER TABLE app_config ENABLE ROW LEVEL SECURITY;
ALTER TABLE app_users ENABLE ROW LEVEL SECURITY;

-- 管理员拥有全部权限
CREATE POLICY "admin_all_exams" ON exams
  FOR ALL USING (
    EXISTS (SELECT 1 FROM app_users WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "admin_all_messages" ON exam_messages
  FOR ALL USING (
    EXISTS (SELECT 1 FROM app_users WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "admin_all_scores" ON scenario_scores
  FOR ALL USING (
    EXISTS (SELECT 1 FROM app_users WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "admin_all_metrics" ON exam_metrics
  FOR ALL USING (
    EXISTS (SELECT 1 FROM app_users WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "admin_all_config" ON app_config
  FOR ALL USING (
    EXISTS (SELECT 1 FROM app_users WHERE user_id = auth.uid() AND role = 'admin')
  );

CREATE POLICY "admin_all_app_users" ON app_users
  FOR ALL USING (
    EXISTS (SELECT 1 FROM app_users WHERE user_id = auth.uid() AND role = 'admin')
  );

-- 考生只能看自己的数据
CREATE POLICY "examinee_own_exams" ON exams
  FOR SELECT USING (user_id = auth.uid());

CREATE POLICY "examinee_own_messages" ON exam_messages
  FOR SELECT USING (
    exam_id IN (SELECT id FROM exams WHERE user_id = auth.uid())
  );

CREATE POLICY "examinee_own_scores" ON scenario_scores
  FOR SELECT USING (
    exam_id IN (SELECT id FROM exams WHERE user_id = auth.uid())
  );

CREATE POLICY "examinee_own_metrics" ON exam_metrics
  FOR SELECT USING (
    exam_id IN (SELECT id FROM exams WHERE user_id = auth.uid())
  );

-- app_config 和 app_users 对考生只读（读取 LLM 配置）
CREATE POLICY "examinee_read_config" ON app_config
  FOR SELECT USING (true);

CREATE POLICY "examinee_read_own_app_user" ON app_users
  FOR SELECT USING (user_id = auth.uid());

-- 允许匿名/新用户注册后插入自己的 app_users 记录
CREATE POLICY "allow_insert_app_users" ON app_users
  FOR INSERT WITH CHECK (user_id = auth.uid());
```

## 4. 获取 API 信息

1. 点击左侧 **Project Settings** → **API**
2. 复制以下两个值：
   - **Project URL**：例如 `https://xxxxxxxxxxxxxxxxxxxx.supabase.co`
   - **anon public**：例如 `eyJhbGciOiJIUzI1NiIs...`

## 5. 填入前端代码

把下面两行替换成你的真实信息：

```javascript
const SUPABASE_URL = 'https://xxxxxxxxxxxxxxxxxxxx.supabase.co';
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIs...';
```

位置在：`/workspace/sales-training-exam/github-pages/index.html` 第 421-422 行。

## 6. 注册首个管理员

1. 部署后打开网站
2. 选择"我是管理员"
3. 输入你的邮箱和密码
4. 点击"注册/登录管理员"
5. 系统会自动在 `app_users` 表中把你标记为 `admin`

## 7. 创建考生账号

1. 以管理员身份进入后台
2. 在"创建考生账号"区域输入考生邮箱和姓名
3. 点击"生成并创建"
4. 系统会生成随机密码，请复制保存并发送给考生

## 8. 配置大模型

1. 在管理后台的"大模型配置"区域
2. 选择 Provider（DeepSeek / OpenAI / 智谱）
3. 输入 API Key
4. 点击"保存配置"
5. 所有考生打开网站都会自动使用这个配置

## 注意事项

- `anon key` 是公开的，数据安全完全依赖 RLS 策略
- 不要暴露 `service_role key`
- 免费版每月 50K 活跃用户、500MB 数据库，对于几十人的培训系统完全够用
