

주제 

- 기업 환경에서 AI 에이전트 간의 통신과 MCP 서버 호출 시 보안과 IAM을 어떻게 구성할지 설명한다


```
┌─────────────────────────────────┐
│  VSCode (MCP Host)              │
│  ┌───────────────────────────┐  │
│  │ Claude Agent              │  │ ← 에이전트
│  │ "please search file system|  |
|  | from github "             │  │
│  └───────┬───────────────────┘  │
│          │ Call MCP Tool        │
└──────────┼──────────────-───────┘
           │
    ┌──────▼──────┐
    │ GitHub MCP  │ ← MCP Server
    │   Server    │
    └─────────────┘
```


Agent-to-Agent 통신

```
┌─────────────────┐         ┌─────────────────┐
│  Coding Agent   │◄───────►│  Research Agent │
│  (VSCode)       │         │  (Browser)      │
└────────┬────────┘         └────────┬────────┘
         │                           │
         │        MCP Servers        │
         └───────────┬───────────────┘
                     │
         ┌───────────▼───────────┐
         │  Shared MCP Servers   │
         │  - GitHub             │
         │  - Slack              │
         │  - Database           │
         └───────────────────────┘
```


- SSO 로그인 및 사용자 식별
- 토큰 검사
- 대리 수행 권한 설정
- 토큰 교환 메커니즘
- 에이전트간 통신 흐름 
- 암호학적 인과간계 

참고자료
https://blog.christianposta.com/


https://blog.christianposta.com/implementing-mcp-dynamic-client-registration-with-spiffe/

https://blog.christianposta.com/authenticating-mcp-oauth-clients-with-spiffe/



https://www.youtube.com/watch?v=uvmzsQMmAp8

