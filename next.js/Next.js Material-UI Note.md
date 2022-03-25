###### tags: `Frontend`

# Next.js Material-UI Note


## Learn source

* Create A Simple Production Website With Material-UI v5 & Nextjs.
    * https://www.ansonlowzf.com/create-a-website-with-material-ui-v5-nextjs/
* Nextjs-mui5 (offcial example, without lint)
    * https://github.com/mui-org/material-ui/tree/master/examples/nextjs


## Initial

create-next-app

```
npx create-next-app mui5-next
cd mui5-next
npm install @mui/material@next @mui/icons-material@next @emotion/cache @emotion/react @emotion/server @emotion/styled 
```

### pages/_app.js

```
import Head from 'next/head';
import { ThemeProvider } from '@mui/material/styles';
import CssBaseline from '@mui/material/CssBaseline';
import { CacheProvider } from '@emotion/react';
import createEmotionCache from '../styles/createEmotionCache';

import "../styles/globals.css";
import theme from '../styles/theme';


// Client-side cache, shared for the whole session of the user in the browser.
const clientSideEmotionCache = createEmotionCache();

export default function MyApp(props) {
  const { Component, emotionCache = clientSideEmotionCache, pageProps } = props;

  return (
    <CacheProvider value={emotionCache}>
      <Head>
        <title>MUI5 Nextjs</title>
        <meta name="viewport" content="initial-scale=1, width=device-width" />
      </Head>
      <ThemeProvider theme={theme}>
        {/* CssBaseline kickstart an elegant, consistent, and simple baseline to build upon. */}
        <CssBaseline />
        <Component {...pageProps} />
      </ThemeProvider>
    </CacheProvider>
  );
}
```

### styles/createEmotionCache.js

```
import createCache from '@emotion/cache';

export default function createEmotionCache() {
  return createCache({ key: 'css' });
}
```

### pages/_document.js

```
import * as React from 'react';
import Document, { Html, Head, Main, NextScript } from 'next/document';
import createEmotionServer from '@emotion/server/create-instance';
import createEmotionCache from '../styles/createEmotionCache';

import theme from '../styles/theme';

export default class MyDocument extends Document {
  render() {
    return (
      <Html lang="en">
        <Head>
          {/* PWA primary color */}
          <meta name="theme-color" content={theme.palette.primary.main} />
          <link
            rel="stylesheet"
            href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap"
          />
        </Head>
        <body>
          <Main />
          <NextScript />
        </body>
      </Html>
    );
  }
}

// `getInitialProps` belongs to `_document` (instead of `_app`),
// it's compatible with static-site generation (SSG).
MyDocument.getInitialProps = async (ctx) => {
  // Resolution order
  //
  // On the server:
  // 1. app.getInitialProps
  // 2. page.getInitialProps
  // 3. document.getInitialProps
  // 4. app.render
  // 5. page.render
  // 6. document.render
  //
  // On the server with error:
  // 1. document.getInitialProps
  // 2. app.render
  // 3. page.render
  // 4. document.render
  //
  // On the client
  // 1. app.getInitialProps
  // 2. page.getInitialProps
  // 3. app.render
  // 4. page.render

  const originalRenderPage = ctx.renderPage;

  // You can consider sharing the same emotion cache between all the SSR requests to speed up performance.
  // However, be aware that it can have global side effects.
  const cache = createEmotionCache();
  const { extractCriticalToChunks } = createEmotionServer(cache);

  ctx.renderPage = () =>
    originalRenderPage({
      enhanceApp: (App) => (props) => <App emotionCache={cache} {...props} />,
    });

  const initialProps = await Document.getInitialProps(ctx);
  // This is important. It prevents emotion to render invalid HTML.
  // See https://github.com/mui-org/material-ui/issues/26561#issuecomment-855286153
  const emotionStyles = extractCriticalToChunks(initialProps.html);
  const emotionStyleTags = emotionStyles.styles.map((style) => (
    <style
      data-emotion={`${style.key} ${style.ids.join(' ')}`}
      key={style.key}
      // eslint-disable-next-line react/no-danger
      dangerouslySetInnerHTML={{ __html: style.css }}
    />
  ));

  return {
    ...initialProps,
    // Styles fragment is rendered after the app and page rendering finish.
    styles: [...React.Children.toArray(initialProps.styles), ...emotionStyleTags],
  };
};
```

### styles/theme.js

```
import { createTheme, responsiveFontSizes } from "@mui/material/styles";
import { deepPurple, amber } from "@mui/material/colors";

// Create a theme instance.
let theme = createTheme({
  palette: {
    primary: deepPurple,
    secondary: amber,
  },
});

theme = responsiveFontSizes(theme);

export default theme;
```

### pages/index.js

```
import Container from "@mui/material/Container";

const Homepage = () => {
  return (
    <Container maxWidth="sm">
    <h1>Home Page</h1>
    <p>lorem*15</p>
    </Container>
  );
};

export default Homepage;
```

### rm file

```
rm styles/Home.module.css
rm pages/api/hello.js
```

### clsx

```
yarn add clsx
```

### components/MuiNextLink.jsx

```
/* eslint-disable jsx-a11y/anchor-has-content */
import * as React from 'react';
import clsx from 'clsx';
import { useRouter } from 'next/router';
import NextLink from 'next/link';
import MuiLink from '@mui/material/Link';

export const NextLinkComposed = React.forwardRef(function NextLinkComposed(props, ref) {
  const { to, linkAs, href, replace, scroll, passHref, shallow, prefetch, locale, ...other } =
    props;

  return (
    <NextLink
      href={to}
      prefetch={prefetch}
      as={linkAs}
      replace={replace}
      scroll={scroll}
      shallow={shallow}
      passHref={passHref}
      locale={locale}
    >
      <a ref={ref} {...other} />
    </NextLink>
  );
});

// A styled version of the Next.js Link component:
// https://nextjs.org/docs/#with-link
const Link = React.forwardRef(function Link(props, ref) {
  const {
    activeClassName = 'active',
    as: linkAs,
    className: classNameProps,
    href,
    noLinkStyle,
    role, // Link don't have roles.
    ...other
  } = props;

  const router = useRouter();
  const pathname = typeof href === 'string' ? href : href.pathname;
  const className = clsx(classNameProps, {
    [activeClassName]: router.pathname === pathname && activeClassName,
  });

  const isExternal =
    typeof href === 'string' && (href.indexOf('http') === 0 || href.indexOf('mailto:') === 0);

  if (isExternal) {
    if (noLinkStyle) {
      return <a className={className} href={href} ref={ref} {...other} />;
    }

    return <MuiLink className={className} href={href} ref={ref} {...other} />;
  }

  if (noLinkStyle) {
    return <NextLinkComposed className={className} ref={ref} to={href} {...other} />;
  }

  return (
    <MuiLink
      component={NextLinkComposed}
      linkAs={linkAs}
      className={className}
      ref={ref}
      to={href}
      {...other}
    />
  );
});

export default Link;
```

### styles/global.css

```
.active {
  opacity: 1;
}
```

### prettier

```
yarn add --dev eslint-config-prettier
```

.eslintrc.json

```
{
  "extends": ["next/core-web-vitals", "prettier"]
}
```

## Others

### windows 開browser 熱更新

```
"dev": "start http://localhost:3000 && next dev",
```

### Cannot find module 'next/babel'

追加.babelrc會導致不使用新版SWC編譯器(Rust編譯器)

.babelrc
```
{
  "presets": ["next/babel"],
  "plugins": []
}
```

.eslintrc
```
{
  "extends": ["next/babel"]
}
```

### .env

local以外NODE_ENV都是production

```
NODE_ENV=development
PORT=3000
ENV_VARIABLE="server_only_variable"
NEXT_PUBLIC_ENV_VARIABLE="public_variable"

```

### Dockerfile

```
# Install dependencies only when needed
FROM node:14-alpine AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Rebuild the source code only when needed
FROM node:14-alpine AS builder
WORKDIR /app
COPY . .
COPY --from=deps /app/node_modules ./node_modules
RUN yarn build

# Production image, copy all the files and run next
FROM node:14-alpine AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# You only need to copy next.config.js if you are NOT using the default configuration
# COPY --from=builder /app/next.config.js ./
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

USER nextjs

EXPOSE 3000

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry.
# ENV NEXT_TELEMETRY_DISABLED 1

CMD ["yarn", "start"]
```

### Lighthouse

網頁質量評估工具

* https://developers.google.com/web/tools/lighthouse