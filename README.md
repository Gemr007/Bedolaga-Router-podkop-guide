# Тариф "Роутер" с Podkop-гайдом для bedolaga-cabinet

Вместо стандартной страницы установки пользователи с роутерным тарифом видят свои VLESS-ключи и инструкцию по настройке [Podkop](https://github.com/itdoginfo/podkop) на OpenWRT.

![Пример](https://github.com/Gemr007/Bedolaga-Router-podkop-guide/blob/main/Example.png?raw=true)

---

## Установка

### Шаг 1. Добавь эндпоинт в бот

```bash
cat >> ~/remnawave-bedolaga-telegram-bot/app/cabinet/routes/subscription.py << 'EOF'


import base64
import httpx
from fastapi import HTTPException
from fastapi.responses import JSONResponse

@router.get('/vless-keys')
async def get_vless_keys(
    user=Depends(get_current_cabinet_user),
    db: AsyncSession = Depends(get_cabinet_db),
    subscription_id: int | None = Query(None),
):
    from ...database.models import Subscription
    from sqlalchemy import select
    query = select(Subscription).where(Subscription.user_id == user.id)
    if subscription_id:
        query = query.where(Subscription.id == subscription_id)
    result = await db.execute(query)
    subscription = result.scalar_one_or_none()
    if not subscription or not subscription.subscription_url:
        raise HTTPException(status_code=404, detail='Subscription not found')
    async with httpx.AsyncClient() as client:
        resp = await client.get(subscription.subscription_url, timeout=10)
    try:
        decoded = base64.b64decode(resp.text.strip()).decode('utf-8')
    except Exception:
        decoded = resp.text
    keys = [l.strip() for l in decoded.split('\n') if l.strip().startswith(('vless://', 'vmess://', 'ss://'))]
    return JSONResponse({'keys': keys})
EOF
```

Перезапусти бота:

```bash
cd ~/remnawave-bedolaga-telegram-bot
make reload
```

---

### Шаг 2. Установи файлы компонентов

#### Вариант А — скачать с GitHub

```bash
cd ~/remnawave-bedolaga-telegram-bot/bedolaga-cabinet/src

curl -o components/connection/PodkopGuide.tsx \
  https://raw.githubusercontent.com/Gemr007/Bedolaga-Router-podkop-guide/main/PodkopGuide.tsx

curl -o pages/Connection.tsx \
  https://raw.githubusercontent.com/Gemr007/Bedolaga-Router-podkop-guide/main/Connection.tsx
```

Проверь:

```bash
ls components/connection/PodkopGuide.tsx pages/Connection.tsx
```

---

#### Вариант Б — собрать вручную

**2.1. Создай компонент PodkopGuide**

```bash
cat > ~/remnawave-bedolaga-telegram-bot/bedolaga-cabinet/src/components/connection/PodkopGuide.tsx << 'EOF'
import { useState, useCallback, useEffect } from 'react';
import apiClient from '../../api/client';

interface Props {
  subscriptionUrl: string | null;
  subscriptionId?: number;
  onGoBack: () => void;
  isTelegramWebApp: boolean;
}

const CheckIcon = () => (
  <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={2.5}>
    <path strokeLinecap="round" strokeLinejoin="round" d="M4.5 12.75l6 6 9-13.5" />
  </svg>
);

const CopyIcon = () => (
  <svg className="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={1.5}>
    <path strokeLinecap="round" strokeLinejoin="round"
      d="M15.75 17.25v3.375c0 .621-.504 1.125-1.125 1.125h-9.75a1.125 1.125 0 01-1.125-1.125V7.875c0-.621.504-1.125 1.125-1.125H6.75a9.06 9.06 0 011.5.124m7.5 10.376h3.375c.621 0 1.125-.504 1.125-1.125V11.25c0-4.46-3.243-8.161-7.5-8.876a9.06 9.06 0 00-1.5-.124H9.375c-.621 0-1.125.504-1.125 1.125v3.5m7.5 10.375H9.375a1.125 1.125 0 01-1.125-1.125v-9.25m12 6.625v-1.875a3.375 3.375 0 00-3.375-3.375h-1.5a1.125 1.125 0 01-1.125-1.125v-1.5a3.375 3.375 0 00-3.375-3.375H9.75"
    />
  </svg>
);

const BackIcon = () => (
  <svg className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={2}>
    <path strokeLinecap="round" strokeLinejoin="round" d="M15 19l-7-7 7-7" />
  </svg>
);

const RouterIcon = () => (
  <svg className="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" strokeWidth={1.5}>
    <path strokeLinecap="round" strokeLinejoin="round"
      d="M8.288 15.038a5.25 5.25 0 017.424 0M5.106 11.856c3.807-3.808 9.98-3.808 13.788 0M1.924 8.674c5.565-5.565 14.587-5.565 20.152 0M12.53 18.22l-.53.53-.53-.53a.75.75 0 011.06 0z"
    />
  </svg>
);

const Step = ({ num, title, children }: { num: number; title: string; children: React.ReactNode }) => (
  <div className="flex gap-3">
    <div className="flex h-7 w-7 shrink-0 items-center justify-center rounded-full bg-accent-500/15 text-sm font-bold text-accent-400 ring-1 ring-accent-500/30">
      {num}
    </div>
    <div className="flex flex-col gap-1 pb-1">
      <p className="text-sm font-semibold text-dark-100">{title}</p>
      <div className="text-xs leading-relaxed text-dark-400">{children}</div>
    </div>
  </div>
);

function parseKeyName(uri: string): string {
  try {
    const hash = uri.split('#')[1];
    if (!hash) return 'Сервер';
    return decodeURIComponent(hash).replace(/[\u{1F300}-\u{1FFFF}]/gu, '').trim() || 'Сервер';
  } catch {
    return 'Сервер';
  }
}

export default function PodkopGuide({ subscriptionUrl, subscriptionId, onGoBack, isTelegramWebApp }: Props) {
  const [vlessKeys, setVlessKeys] = useState<string[]>([]);
  const [loading, setLoading] = useState(false);
  const [copiedIndex, setCopiedIndex] = useState<number | null>(null);

  useEffect(() => {
    if (!subscriptionId && !subscriptionUrl) return;
    setLoading(true);
    const url = subscriptionId
      ? `/cabinet/subscription/vless-keys?subscription_id=${subscriptionId}`
      : `/cabinet/subscription/vless-keys`;
    apiClient.get(url)
      .then((response) => { const data = response.data; setVlessKeys(data.keys || []); })
      .catch(() => setVlessKeys([]))
      .finally(() => setLoading(false));
  }, [subscriptionId, subscriptionUrl]);

  const handleCopy = useCallback((key: string, index: number) => {
    navigator.clipboard.writeText(key).then(() => {
      setCopiedIndex(index);
      setTimeout(() => setCopiedIndex(null), 2500);
    });
  }, []);

  return (
    <div className="space-y-5 pb-6">
      <div className="flex items-center gap-3">
        {!isTelegramWebApp && (
          <button onClick={onGoBack} className="flex h-10 w-10 items-center justify-center rounded-xl border border-dark-700 bg-dark-800 transition-colors hover:border-dark-600">
            <BackIcon />
          </button>
        )}
        <div className="flex flex-1 items-center gap-2">
          <span className="text-dark-300"><RouterIcon /></span>
          <h2 className="text-lg font-bold text-dark-100">Настройка роутера</h2>
        </div>
        <span className="rounded-lg bg-accent-500/10 px-2.5 py-1 text-xs font-medium text-accent-400 ring-1 ring-accent-500/20">OpenWRT</span>
      </div>

      <div className="rounded-2xl border border-dark-700/60 bg-dark-800/60 p-4">
        <p className="mb-3 text-xs font-medium uppercase tracking-wider text-dark-400">Ваши VLESS-ключи</p>
        {loading && (
          <div className="flex items-center justify-center py-6">
            <div className="h-6 w-6 animate-spin rounded-full border-2 border-accent-500/30 border-t-accent-500" />
          </div>
        )}
        {!loading && vlessKeys.length === 0 && <p className="text-xs italic text-dark-500">Ключи недоступны</p>}
        {!loading && vlessKeys.length > 0 && (
          <div className="flex flex-col gap-2">
            {vlessKeys.map((key, i) => {
              const isCopied = copiedIndex === i;
              return (
                <div key={i} className="rounded-xl border border-dark-700/50 bg-dark-900/60 p-3">
                  <div className="mb-2 flex items-center justify-between gap-2">
                    <span className="truncate text-xs font-medium text-dark-300">{parseKeyName(key)}</span>
                    <button
                      onClick={() => handleCopy(key, i)}
                      className={`flex shrink-0 items-center gap-1.5 rounded-lg px-2.5 py-1 text-xs font-medium transition-all ${isCopied ? 'bg-green-600/20 text-green-400 ring-1 ring-green-500/30' : 'bg-accent-500/10 text-accent-400 ring-1 ring-accent-500/20 hover:bg-accent-500/20'}`}
                    >
                      {isCopied ? <CheckIcon /> : <CopyIcon />}
                      {isCopied ? 'Скопировано' : 'Копировать'}
                    </button>
                  </div>
                  <p className="break-all font-mono text-[10px] leading-relaxed text-dark-500">
                    {key.length > 80 ? key.slice(0, 80) + '…' : key}
                  </p>
                </div>
              );
            })}
          </div>
        )}
      </div>

      <div className="rounded-2xl border border-dark-700/60 bg-dark-800/40 p-4">
        <p className="mb-4 text-xs font-medium uppercase tracking-wider text-dark-400">Инструкция по настройке Podkop</p>
        <div className="flex flex-col gap-4">
          <Step num={1} title="Установите пакет Podkop">
            Подключитесь к роутеру по SSH и выполните:
            <code className="mt-1.5 block rounded-lg bg-dark-900/80 px-3 py-2 font-mono text-xs text-dark-300 ring-1 ring-dark-700/50">
              opkg update && opkg install podkop
            </code>
          </Step>
          <div className="h-px bg-dark-700/40" />
          <Step num={2} title="Откройте настройки Podkop">
            В веб-интерфейсе LuCI перейдите:
            <span className="mt-1 inline-flex items-center gap-1 text-dark-300">
              <span className="rounded bg-dark-900/80 px-1.5 py-0.5 font-mono text-xs ring-1 ring-dark-700/50">Сервисы</span>
              {' → '}
              <span className="rounded bg-dark-900/80 px-1.5 py-0.5 font-mono text-xs ring-1 ring-dark-700/50">Podkop</span>
            </span>
          </Step>
          <div className="h-px bg-dark-700/40" />
          <Step num={3} title="Вставьте VLESS-ключ">
            Скопируйте нужный ключ выше. В поле{' '}
            <span className="rounded bg-dark-900/80 px-1.5 py-0.5 font-mono text-xs text-dark-300 ring-1 ring-dark-700/50">Outbound</span>{' '}
            выберите тип <b className="text-dark-200">VLESS URI</b> и вставьте ключ.
          </Step>
          <div className="h-px bg-dark-700/40" />
          <Step num={4} title="Выберите режим работы">
            Рекомендуем <b className="text-dark-200">Proxy only blocked</b> — трафик на заблокированные сайты пойдёт через VPN, остальной — напрямую.
          </Step>
          <div className="h-px bg-dark-700/40" />
          <Step num={5} title="Сохраните настройки">
            Нажмите <b className="text-dark-200">Save & Apply</b>. Podkop автоматически запустится.
          </Step>
        </div>
      </div>

      <a href="https://github.com/itdoginfo/podkop" target="_blank" rel="noopener noreferrer"
        className="flex items-center justify-center gap-2 rounded-xl border border-dark-700/60 bg-dark-800/40 px-4 py-3 text-sm text-dark-300 transition-colors hover:border-dark-600 hover:text-dark-100">
        <svg className="h-4 w-4" fill="currentColor" viewBox="0 0 24 24">
          <path d="M12 0C5.37 0 0 5.373 0 12c0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 21.795 24 17.298 24 12c0-6.627-5.373-12-12-12" />
        </svg>
        Документация Podkop на GitHub
      </a>
    </div>
  );
}
EOF
```

**2.2. Модифицируй Connection.tsx**

```bash
python3 << 'PYEOF'
with open('/root/remnawave-bedolaga-telegram-bot/bedolaga-cabinet/src/pages/Connection.tsx', 'r') as f:
    content = f.read()

# Add PodkopGuide import
content = content.replace(
    "import InstallationGuide from '../components/connection/InstallationGuide';",
    "import InstallationGuide from '../components/connection/InstallationGuide';\nimport PodkopGuide from '../components/connection/PodkopGuide';"
)

# Add helper function before component
helper = """
function isRouterTariff(name?: string | null): boolean {
  return /router|\u0440\u043e\u0443\u0442\u0435\u0440|openwrt|podkop/i.test(name || '');
}

"""
content = content.replace('export default function Connection()', helper + 'export default function Connection()')

# Add subscription query
subscription_query = """
  const { data: subscriptionStatus } = useQuery({
    queryKey: ['subscription', subId],
    queryFn: () => subscriptionApi.getSubscription(subId),
    staleTime: 30_000,
  });
  const routerTariff = isRouterTariff(subscriptionStatus?.subscription?.tariff_name);
"""
content = content.replace(
    '  const qrConnectionUrl = useMemo(',
    subscription_query + '  const qrConnectionUrl = useMemo('
)

# Add PodkopGuide render
podkop_render = """  if (routerTariff) {
    return (
      <PodkopGuide
        subscriptionUrl={connectionLink?.subscription_url ?? appConfig?.subscriptionUrl ?? null}
        subscriptionId={subId ?? undefined}
        onGoBack={handleGoBack}
        isTelegramWebApp={isTelegramWebApp}
      />
    );
  }

"""
content = content.replace(
    '  if (isLoading || isConnectionLinkLoading)',
    podkop_render + '  if (isLoading || isConnectionLinkLoading)'
)

with open('/root/remnawave-bedolaga-telegram-bot/bedolaga-cabinet/src/pages/Connection.tsx', 'w') as f:
    f.write(content)
print("Done")
PYEOF
```

---

### Шаг 3. Пересобери cabinet

```bash
cd ~/remnawave-bedolaga-telegram-bot/bedolaga-cabinet
docker compose -f docker-compose.local.yml build --no-cache
docker compose -f docker-compose.local.yml up -d --force-recreate
```

---

### Шаг 4. Создай тариф в админке

В панели администратора бота создай тариф с именем содержащим одно из слов:

- `Router` / `router`
- `Роутер` / `роутер`
- `OpenWRT` / `openwrt`
- `Podkop` / `podkop`

Например: **"Router 30 дней"**

---

## Как это работает

1. При открытии страницы установки `Connection.tsx` проверяет `tariff_name` подписки
2. Если имя тарифа совпадает — рендерит `PodkopGuide` вместо стандартного `InstallationGuide`
3. `PodkopGuide` запрашивает `/cabinet/subscription/vless-keys` через авторизованный `apiClient`
4. Бот получает `subscription_url` из БД, делает запрос к Remnawave, декодирует base64 и возвращает список VLESS-ключей
5. Пользователь видит ключи по серверам с кнопкой "Копировать" и инструкцию по Podkop
