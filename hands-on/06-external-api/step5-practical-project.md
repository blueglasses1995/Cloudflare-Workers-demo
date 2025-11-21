# Step 5: 実践プロジェクト - 天気情報API集約サービス

## プロジェクト概要

複数の天気APIを統合し、キャッシング、エラーハンドリング、フォールバックを実装します。

## 完全な実装例

```typescript
interface Env {
  WEATHER_CACHE: KVNamespace;
}

interface WeatherData {
  location: string;
  temperature: number;
  condition: string;
  humidity: number;
  source: string;
  timestamp: string;
}

class WeatherAPI {
  private cache: KVNamespace;

  constructor(cache: KVNamespace) {
    this.cache = cache;
  }

  async getWeather(city: string): Promise<WeatherData> {
    const cacheKey = `weather:${city}`;

    // キャッシュチェック
    const cached = await this.cache.get(cacheKey, { type: 'json' });
    if (cached) {
      return cached as WeatherData;
    }

    // API呼び出し（リトライ付き）
    const data = await this.fetchWithRetry(city);

    // キャッシュに保存（10分）
    await this.cache.put(cacheKey, JSON.stringify(data), {
      expirationTtl: 600,
    });

    return data;
  }

  private async fetchWithRetry(city: string, retries: number = 3): Promise<WeatherData> {
    for (let i = 0; i < retries; i++) {
      try {
        return await this.fetchWeatherData(city);
      } catch (error) {
        if (i === retries - 1) throw error;
        await new Promise((resolve) => setTimeout(resolve, 1000 * (i + 1)));
      }
    }
    throw new Error('Failed to fetch weather data');
  }

  private async fetchWeatherData(city: string): Promise<WeatherData> {
    const lat = 35.6762; // Tokyo
    const lon = 139.6503;

    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current_weather=true`;

    const response = await fetch(url);

    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }

    const data = await response.json();

    return {
      location: city,
      temperature: data.current_weather.temperature,
      condition: this.getCondition(data.current_weather.weathercode),
      humidity: 65, // Open-Meteo doesn't provide this in free tier
      source: 'Open-Meteo',
      timestamp: new Date().toISOString(),
    };
  }

  private getCondition(code: number): string {
    const conditions: Record<number, string> = {
      0: 'Clear',
      1: 'Partly Cloudy',
      2: 'Cloudy',
      3: 'Overcast',
      45: 'Foggy',
      48: 'Foggy',
      51: 'Light Drizzle',
      61: 'Rain',
      80: 'Rain Showers',
      95: 'Thunderstorm',
    };

    return conditions[code] || 'Unknown';
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const city = url.searchParams.get('city') || 'Tokyo';

    try {
      const weatherAPI = new WeatherAPI(env.WEATHER_CACHE);
      const weather = await weatherAPI.getWeather(city);

      return new Response(JSON.stringify(weather, null, 2), {
        headers: {
          'Content-Type': 'application/json',
          'Cache-Control': 'public, max-age=600',
        },
      });
    } catch (error) {
      return new Response(
        JSON.stringify({
          error: 'Failed to fetch weather data',
          message: error instanceof Error ? error.message : 'Unknown error',
        }),
        {
          status: 500,
          headers: { 'Content-Type': 'application/json' },
        }
      );
    }
  },
};
```

## まとめ

このハンズオンで学んだこと：

✅ 外部APIの基本的な呼び出し
✅ エラーハンドリングとリトライ戦略
✅ キャッシングによるパフォーマンス最適化
✅ タイムアウトと並列処理
✅ 本番環境に対応した実装

👉 [ハンズオン07: 認証の実装](../07-authentication/README.md)
