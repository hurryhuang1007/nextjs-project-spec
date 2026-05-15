# fetchApi 封装模板

将以下实现置于 `src/util/fetchApi.ts`（或项目约定路径），并按项目调整 `errorCb`、鉴权 header 等逻辑。

```typescript

// import { revalidateTag } from 'next/cache';
import { getApiRootUrl } from '@/config/url';
import { isProd } from './env';

export interface FetchApiOptions extends RequestInit {
  params?: Record<string, string | number | null | undefined>;
  data?: Record<string, any>;
}

export interface FetchApiExtraOptions<T> {
  baseUrl?: string;
  notJson?: boolean;
  full?: boolean;
  errorCb?: (json: any, fetchApiParams: FetchApiParams<T>) => void;
  onSSE?: (data: T) => void;
  successCode?: number[];
  errorAlertCode?: number[];
  errorAlertMsg?: string | Record<string, string>;
}

export interface FetchApiParams<T> {
  url: string;
  options?: FetchApiOptions;
  extraOptions: FetchApiExtraOptions<T>;
}

export function fetchApiDefaultErrorCb<T>(
  json: any,
  { url, options, extraOptions }: FetchApiParams<T>
) {
  // 可以在这里统一处理错误
  // console.error(json)

  const { errorAlertCode, errorAlertMsg } = extraOptions;
  if (errorAlertCode?.includes(json.code)) {
    // 进行错误提示
    let msg = '';
    if (errorAlertMsg) {
      if (typeof errorAlertMsg === 'string') {
        msg = errorAlertMsg;
      } else if (typeof errorAlertMsg === 'object') {
        msg = errorAlertMsg[json.code];
      }
    }

    if (!msg) {
      msg = json.msg;
    }

    if (msg) {
      alert(msg);
    }
  }

  // ...
}

export type FetchApi<T = unknown> = Promise<T> & { abort: () => void };

export default function fetchApi<T>(
  url: string,
  options?: FetchApiOptions,
  extraOptions?: FetchApiExtraOptions<T>
): FetchApi<T> {
  const {
    baseUrl = getApiRootUrl(),
    notJson = false,
    full = false,
    // 这里传入的函数的作用是错误的通用（常规）处理，特定某个api的错误应使用.catch处理。
    // 此处的参数只是为了增加灵活性
    errorCb = fetchApiDefaultErrorCb,
    onSSE,
    successCode = [1],
    errorAlertCode = [999],
    errorAlertMsg, // 错误提示消息，map时为错误码与提示消息的映射关系，空则使用 res.msg 中的内容
  } = extraOptions || {};

  // 将默认值回填至extraOptions，方便后续传给errorCb
  extraOptions = {
    baseUrl,
    notJson,
    full,
    errorCb,
    onSSE,
    successCode,
    errorAlertCode,
    errorAlertMsg,
  };
  // console.log('fetchApi', url, options, extraOptions);

  let _url = url;
  const _options: RequestInit = { ...options };

  if (options?.params) {
    const params = new URLSearchParams();
    Object.entries(options.params).forEach(([key, value]) => {
      if (value === '' || value == null) return;
      params.append(key, value as string);
    });
    _url = `${url}?${params.toString()}`;
  }

  if (options?.data) {
    _options.body = JSON.stringify(options.data);
    _options.headers = {
      ..._options.headers,
      'Content-Type': 'application/json',
    };
  }
  // _options.headers = {
  //   Authorization: cookieGet('token') || cookieGet('shutu_access_token'),
  //   'X-Device-Type': getIsServer() ? '4' : (await getDeviceType()) + '',
  //   ..._options.headers,
  // };
  // if (!(_options.headers as any).Authorization) {
  //   delete (_options.headers as any).Authorization;
  // }

  if (_options.next?.tags) {
    _options.next.tags.push('__api__');
  } else if (_options.next) {
    _options.next.tags = ['__api__'];
  } else {
    _options.next = { tags: ['__api__'] };
  }

  if (isProd) {
    if (_options.next.revalidate == undefined) {
      // 默认缓存 10min
      _options.next.revalidate = 60 * 10;
    }
  } else {
    // 开发及测试环境关闭缓存
    _options.cache = 'no-store';
  }

  let abortController: AbortController | undefined;
  if (typeof AbortController !== 'undefined') {
    abortController = new AbortController();
    _options.signal = abortController.signal;
  }

  let finished = false;
  const promise = fetch(baseUrl + _url, {
    ..._options,
    mode: 'cors',
    credentials: 'include',
  }).then(async res => {
    if (onSSE) {
      const reader = res.body!.getReader();
      const decoder = new TextDecoder();

      let lastMessage: T;
      return reader.read().then(function read({ done, value }): Promise<T> | T {
        if (done) {
          finished = true;
          return lastMessage;
        }

        const data = decoder.decode(value);

        const pattern = /data: (.+?)\n\n/g;
        const matches = [];

        let match;
        while ((match = pattern.exec(data)) !== null) {
          matches.push(match[1]);
        }

        matches.forEach(data => {
          if (!data) {
            throw new Error(`sse api: ${url} 发生格式错误: ${data}`);
          }
          if (typeof window !== 'undefined' && (window as any).__debug) {
            console.log('sse data', data);
          }

          let message: T;
          if (notJson) {
            message = data as unknown as T;
          } else if (full) {
            message = JSON.parse(data) as T;
          } else {
            const json = JSON.parse(data);
            if (!successCode.includes(json.code)) {
              errorCb(json, { url, options, extraOptions: extraOptions! });
              const error: Error & { code?: number; data?: any; msg?: string } = new Error(
                `sse api: ${url} 发生错误: ${(json.msg as string) || '未知错误'}`
              );
              error.code = json.code;
              error.data = json.data;
              error.msg = json.msg;
              throw error;
            }
            message = json.data;
          }
          lastMessage = message;
          onSSE(message);
        });

        return reader.read().then(read);
      });
    } else {
      finished = true;
      let result: T;
      if (notJson) {
        result = res as unknown as T;
      } else if (full) {
        result = res.json() as T;
      } else {
        const json = await res.json();
        if (!successCode.includes(json.code)) {
          errorCb(json, { url, options, extraOptions: extraOptions! });
          const error: Error & { code?: number; data?: any; msg?: string } = new Error(
            `请求api: ${url} 发生错误: ${(json.msg as string) || '未知错误'}`
          );
          error.code = json.code;
          error.data = json.data;
          error.msg = json.msg;
          throw error;
        }
        result = json.data;
      }

      return result;
    }
  });

  (promise as FetchApi<T>).abort = () => {
    try {
      if (!finished) {
        abortController!.abort();
      }
    } catch (e) {
      console.warn('abort error', e);
    }
  };
  return promise as FetchApi<T>;
}

// export function clearApiCache() {
//   revalidateTag('__api__');
// }

```

## 使用要点

- 特定 API 的错误用 `.catch` 处理；`errorCb` 仅作可配置的通用处理。
- 生产环境默认 `revalidate` 10 分钟；非生产 `cache: 'no-store'`。
- SSE 通过 `extraOptions.onSSE` 回调逐条消费。
