arena-api-melody

登录相关

```java
src/main/java/me/ele/napos/arena/api/melody/
├── descriptor
│   ├── constraint
│   │   └── Constraint.java
│   ├── service
│   │   └── keeper
│   │       ├── ArenaOAuthService.java
│   │       ├── ChangePasswordService.java
│   │       ├── KeeperService.java
│   │       ├── LoginService.java
│   │       └── ResetPasswordService.java
│   └── struct
│       ├── keeper
│       │   ├── BasicKeeper.java
│       │   ├── LoginFailureData.java
│       │   ├── LoginResult.java
│       │   ├── LoginSuccessData.java
│       │   ├── LoginSuccessShop.java
│       │   ├── ResetKeeper.java
│       │   └── ResetProcess.java
│       ├── login
│       │   └── ChainKeeper.java
│       └── oauth
│           └── AuthorizationStatus.java
└── processor
    ├── ArenaApiMelodyProcessor.java
    ├── actor
    │   ├── ArenaCache.java
    │   └── ArenaConverter.java
    ├── atom
    │   └── SessionKeeper.java
    ├── config
    │   └── SmsSenderKeyConfig.java
    ├── converter
    │   └── KeeperConversion.java
    ├── filter
    │   └── SessionFilter.java
    ├── helper
    │   ├── PasswordHelper.java
    │   ├── VerifyCodeHelper.java
    │   ├── bankCard
    │   │   └── BankCardHelper.java
    │   ├── keeper
    │   │   └── ArenaKeeperHelper.java
    │   ├── marketingManager
    │   │   └── VoteMarketingManagerHelper.java
    │   ├── notice
    │   │   └── NoticeHelper.java
    │   └── order
    │       ├── ApolloOrderHelper.java
    │       └── ExceptionOrderHelper.java
    ├── impl
    │   ├── ArenaOAuthServiceImpl.java
    │   ├── ChangePasswordServiceImpl.java
    │   ├── KeeperServiceImpl.java
    │   ├── LoginServiceImpl.java
    │   └── ResetPasswordServiceImpl.java
    └── util
        ├── Captcha.java
        └── PasswordUtil.java
```

