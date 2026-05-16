// ============================================
// QUOTESNOVA - FINAL PRODUCTION WORKER vFINAL
// All issues resolved. Edge HTML caching. Complete CSP.
// ============================================

export default {
  async fetch(request) {
    const SUPABASE_URL = 'https://cdyiropvfbooczxkllqr.supabase.co';
    const ANON_KEY = 'sb_publishable_yPKnq6TFtOJvvYm9YpFYbQ_rjq7WtF0';
    const SITE_URL = 'https://quotesnova.com';
    const SITE_NAME = 'QuotesNova';
    
    // ============================================
    // RATE LIMITING WITH AUTO-CLEANUP
    // ============================================
    const ip = request.headers.get('cf-connecting-ip') || 'unknown';
    const now = Date.now();
    const WINDOW = 1000;
    if (!globalThis.rateMap) globalThis.rateMap = new Map();
    for (const [key, timestamp] of globalThis.rateMap) {
      if (now - timestamp > WINDOW) globalThis.rateMap.delete(key);
    }
    const lastRequest = globalThis.rateMap.get(ip) || 0;
    if (now - lastRequest < WINDOW) {
      return new Response('Too many requests. Please wait.', { 
        status: 429,
        headers: { 'Retry-After': '1' }
      });
    }
    globalThis.rateMap.set(ip, now);
    
    const url = new URL(request.url);
    const page = Math.max(1, parseInt(url.searchParams.get('page') || '1'));
    const limit = page === 1 ? 6 : 12;
    const pageUrl = page === 1 ? SITE_URL : `${SITE_URL}/?page=${page}`;
    
    // ============================================
    // EDGE HTML CACHING
    // ============================================
    const cache = caches.default;
    const htmlCacheKey = new Request(request.url, request);
    const cachedHTML = await cache.match(htmlCacheKey);
    if (cachedHTML) return cachedHTML;
    
    const seoTitles = [
      "Trending Quotes on QuotesNova",
      "Most Loved Quotes Today", 
      "Popular Inspirational Quotes",
      "Top Quotes This Week",
      "Wisdom & Inspiration Collection"
    ];
    const seoTitle = seoTitles[(page - 1) % seoTitles.length];
    
    try {
      // ============================================
      // PARALLEL FETCH: CSS + Categories + Quotes
      // ============================================
      const [cssResult, catResult, quotesResult] = await Promise.all([
        (async () => {
          const key = 'https://css-cache/quotesnova-main.css';
          let cached = await cache.match(key);
          if (cached) return cached.clone().text();
          const res = await fetch('https://quoteshub.pages.dev/css/main.css');
          if (!res.ok) return 'body{background:#000;color:#fff;font-family:sans-serif}';
          const text = await res.text();
          const resp = new Response(text, {
            headers: { 'Cache-Control': 'public, max-age=86400', 'Content-Type': 'text/css' }
          });
          await cache.put(key, resp.clone());
          return text;
        })(),
        
        (async () => {
          const key = 'https://data-cache/categories';
          let cached = await cache.match(key);
          if (cached) return cached.clone().json();
          const res = await fetch(
            `${SUPABASE_URL}/rest/v1/categories?select=name,slug&is_active=eq.true&limit=8`,
            { headers: { 'apikey': ANON_KEY, 'Authorization': `Bearer ${ANON_KEY}` } }
          );
          const data = res.ok ? await res.json() : [];
          const resp = new Response(JSON.stringify(data), {
            headers: { 'Cache-Control': 'public, max-age=3600', 'Content-Type': 'application/json' }
          });
          await cache.put(key, resp.clone());
          return data;
        })(),
        
        (async () => {
          const res = await fetch(
            `${SUPABASE_URL}/rest/v1/quotes?select=id,quote_text,author_name,media_url,likes_count,user_id,slug,created_at&is_published=eq.true&order=likes_count.desc&limit=${limit}&offset=${(page-1)*limit}`,
            { headers: { 'apikey': ANON_KEY, 'Authorization': `Bearer ${ANON_KEY}` } }
          );
          return res.ok ? await res.json() : [];
        })()
      ]);
      
      const css = cssResult;
      const categories = catResult;
      const quotes = quotesResult;
      
      // ============================================
      // FETCH AUTHORS (quoted UUID format)
      // ============================================
      const userIds = [...new Set((quotes || []).map(q => q.user_id).filter(Boolean))];
      let authorMap = {};
      if (userIds.length > 0) {
        const safeIds = userIds.slice(0, 20).map(id => `"${id}"`).join(',');
        const authorRes = await fetch(
          `${SUPABASE_URL}/rest/v1/authors?select=id,name,profile_image_url,is_verified&id=in.(${safeIds})`,
          { headers: { 'apikey': ANON_KEY, 'Authorization': `Bearer ${ANON_KEY}` } }
        );
        const authors = authorRes.ok ? await authorRes.json() : [];
        authors.forEach(a => { authorMap[a.id] = a; });
      }
      
      // ============================================
      // DYNAMIC DESCRIPTION
      // ============================================
      const topQuote = quotes?.[0]?.quote_text || '';
      const dynamicDesc = topQuote 
        ? `"${topQuote.substring(0, 100)}" – Discover more inspirational quotes on ${SITE_NAME}.`
        : `Discover and share wisdom that moves you. ${SITE_NAME} is home to millions of inspirational quotes.`;
      
      // ============================================
      // BUILD COMPONENTS
      // ============================================
      const categoriesHTML = (categories || []).map(c =>
        `<a href="/category/${c.slug}" class="category-pill">${esc(c.name)}</a>`
      ).join('');
      
      const seoCategoriesHTML = (categories || []).slice(0, 5).map(c =>
        `<a href="/category/${c.slug}" class="seo-link">${esc(c.name)} Quotes</a>`
      ).join('');
      
      const quotesHTML = (quotes || []).map(q => {
        const author = authorMap[q.user_id] || {};
        const isVerified = author.is_verified || false;
        const avatarUrl = author.profile_image_url || '';
        const authorName = q.author_name || 'Anonymous';
        const initial = authorName.charAt(0).toUpperCase();
        const likes = Number(q.likes_count || 0);
        const imgAlt = (q.quote_text || '').substring(0, 80);
        const avatarHTML = avatarUrl
          ? `<img src="${avatarUrl}" class="post-avatar" alt="${esc(authorName)}" loading="lazy">`
          : `<div class="post-avatar" style="background:linear-gradient(135deg,#F86D2A,#e05a1e);display:flex;align-items:center;justify-content:center;color:white;font-weight:600;font-size:16px;">${initial}</div>`;
        const verifiedBadge = isVerified ? '<span class="verified-badge" title="Verified"><i class="fas fa-check-circle"></i></span>' : '';
        const imageHTML = q.media_url
          ? `<div class="post-image-container"><img src="${q.media_url}" class="post-image" loading="lazy" alt="${esc(imgAlt)}"></div>`
          : '<div class="post-image-placeholder"><i class="fas fa-quote-right"></i></div>';
        return `
        <article class="quote-card">
          <a href="/quote/${q.slug}" class="quote-link" data-quote-id="${q.id}">
            <div class="post-header">
              <div class="post-user">
                <div class="post-avatar-wrapper">${avatarHTML}</div>
                <div class="post-info">
                  <div class="post-name"><span class="author-name">${esc(authorName)}</span>${verifiedBadge}</div>
                  <div class="post-time">Featured</div>
                </div>
              </div>
            </div>
            ${imageHTML}
            <div class="post-caption"><span class="caption-text">"${esc(q.quote_text || '')}"</span></div>
            <div class="likes-count">${likes.toLocaleString()} likes</div>
            <div class="view-comments">View quote</div>
          </a>
        </article>`;
      }).join('') || '<div class="empty-state"><p>No quotes yet. <a href="/create">Be the first!</a></p></div>';
      
      const seoLinksHTML = (quotes || []).slice(0, 5).map(q => {
        const words = (q.quote_text || '').split(' ').slice(0, 10).join(' ');
        return `<a href="/quote/${q.slug}" class="seo-link">${esc(words)}...</a>`;
      }).join('');
      
      const paginationHTML = quotes?.length >= limit ? `
      <div class="pagination">
        <a href="/?page=${page + 1}" class="load-more-btn">More Inspirational Quotes →</a>
      </div>` : '';
      
      // ============================================
      // STRUCTURED DATA
      // ============================================
      const itemListSchema = (quotes || []).map((q, i) => ({
        "@type": "ListItem",
        "position": ((page - 1) * limit) + i + 1,
        "url": `${SITE_URL}/quote/${q.slug}`,
        "name": (q.quote_text || '').split(' ').slice(0, 10).join(' ')
      }));
      
      const quoteArticlesSchema = (quotes || []).slice(0, 3).map(q => ({
        "@context": "https://schema.org",
        "@type": "CreativeWork",
        "text": q.quote_text,
        "author": { "@type": "Person", "name": q.author_name || "Anonymous" },
        "url": `${SITE_URL}/quote/${q.slug}`
      }));
      
      const initialState = JSON.stringify({ quotes, categories, authorMap, page }).replace(/</g, '\\u003c');
      
      // ============================================
      // FULL HTML
      // ============================================
      const html = `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<title>${SITE_NAME} · Wisdom &amp; Inspiration${page > 1 ? ` - Page ${page}` : ''}</title>
<meta name="description" content="${esc(dynamicDesc)}">
<meta name="robots" content="index, follow, max-snippet:-1, max-image-preview:large">
<meta name="application-name" content="${SITE_NAME}">
<meta itemprop="name" content="${SITE_NAME}">
<meta name="theme-color" content="#F86D2A">
<meta property="og:type" content="website">
<meta property="og:site_name" content="${SITE_NAME}">
<meta property="og:title" content="${SITE_NAME} · Wisdom &amp; Inspiration for Every Soul">
<meta property="og:description" content="${esc(dynamicDesc)}">
<meta property="og:image" content="${SITE_URL}/og-image.jpg">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:url" content="${pageUrl}">
<meta property="og:locale" content="en_US">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@quotesnovas">
<meta name="twitter:title" content="${SITE_NAME} · Wisdom &amp; Inspiration for Every Soul">
<meta name="twitter:description" content="${esc(dynamicDesc)}">
<meta name="twitter:image" content="${SITE_URL}/og-image.jpg">
<link rel="canonical" href="${pageUrl}">
${page > 1 ? `<link rel="prev" href="${SITE_URL}/?page=${page - 1}">` : ''}
${quotes?.length >= limit ? `<link rel="next" href="${SITE_URL}/?page=${page + 1}">` : ''}
<link rel="icon" type="image/svg+xml" href="/favicon.svg">
<link rel="preload" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" as="style">
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=Playfair+Display:wght@500;600&display=swap" rel="stylesheet">
<style>${css}
.homepage-hero{text-align:center;padding:20px 16px 8px;background:#000}
.homepage-hero h1{font-size:20px;color:#F5F5F5;margin-bottom:4px}
.homepage-hero p{font-size:13px;color:#A8A8A8}
.seo-links{padding:16px;border-top:1px solid #262626;margin-top:16px}
.seo-links h2{font-size:14px;color:#F86D2A;margin-bottom:8px}
.seo-links nav{display:flex;flex-direction:column;gap:4px}
.seo-link{display:block;color:#A8A8A8;text-decoration:none;font-size:12px;padding:6px 0;border-bottom:1px solid #1A1A1A}
.seo-link:hover{color:#F5F5F5}
.quote-card{border-bottom:1px solid #262626}
.quote-link{text-decoration:none;color:inherit;display:block}
.author-name{color:#F5F5F5;text-decoration:none;font-weight:600}
.pagination{text-align:center;padding:20px}
.load-more-btn{display:inline-block;background:#F86D2A;color:#fff;padding:12px 32px;border-radius:30px;text-decoration:none;font-weight:600}
</style>
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebSite",
  "name": "${SITE_NAME}",
  "alternateName": ["Quotes Nova", "QuotesNova App"],
  "url": "${SITE_URL}/",
  "description": "Discover and share wisdom that moves you.",
  "publisher": { "@type": "Organization", "name": "${SITE_NAME}", "url": "${SITE_URL}", "logo": { "@type": "ImageObject", "url": "${SITE_URL}/logo.png", "width": 512, "height": 512 } },
  "potentialAction": { "@type": "SearchAction", "target": "${SITE_URL}/explore?q={search_term_string}", "query-input": "required name=search_term_string" }
}
</script>
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "name": "${SITE_NAME}",
  "url": "${SITE_URL}",
  "logo": "${SITE_URL}/logo.png",
  "description": "QuotesNova is the world's most vibrant quote-sharing community.",
  "foundingDate": "2024",
  "founder": { "@type": "Person", "name": "Gift Kapokola" },
  "sameAs": ["https://x.com/quotesnovas","https://www.instagram.com/quotesnovas","https://www.facebook.com/quotesnova","https://youtube.com/@quotesnova"]
}
</script>
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "ItemList",
  "name": "${seoTitle}",
  "numberOfItems": ${quotes?.length || 0},
  "itemListElement": ${JSON.stringify(itemListSchema)}
}
</script>
${quoteArticlesSchema.map(s => `<script type="application/ld+json">${JSON.stringify(s)}</script>`).join('\n')}
</head>
<body>
<div id="app" data-hydrated="false">
<header class="app-header"><a href="/" class="app-logo" style="text-decoration:none">Quotes<span>Nova</span></a><div class="header-actions"><a href="/explore" aria-label="Search"><i class="fas fa-search"></i></a><a href="/activity" aria-label="Notifications"><i class="far fa-bell"></i></a></div></header>
<section class="homepage-hero"><h1>QuotesNova – Wisdom &amp; Inspiration for Every Soul</h1><p>Discover and share meaningful quotes from authors around the world.</p></section>
<div id="homepage-elements">
<nav class="feed-tabs"><a href="/" class="feed-tab active">For You</a><a href="/?filter=following" class="feed-tab">Following</a></nav>
<section class="categories-row"><div class="categories-title"><i class="fas fa-fire"></i> Trending Categories</div><nav class="categories-scroll">${categoriesHTML}</nav></section>
<section id="static-feed">${quotesHTML}</section>
${paginationHTML}
<section class="seo-links"><h2>Latest Quotes</h2><nav>${seoLinksHTML}</nav></section>
<section class="seo-links"><h2>Popular Categories</h2><nav>${seoCategoriesHTML}</nav></section>
<section class="seo-links"><h2>Discover More</h2><nav><a href="/explore" class="seo-link">Explore All Quotes →</a><a href="/categories" class="seo-link">Browse Categories →</a><a href="/authors" class="seo-link">Meet Our Authors →</a></nav></section>
<div class="static-login-prompt" id="login-prompt" style="display:none;"><h3>Join QuotesNova</h3><p>Create an account to save quotes, follow authors, and share your own wisdom.</p><div><a href="/register" class="static-btn">Sign Up</a><a href="/login" class="static-btn static-btn-outline">Log In</a></div></div>
</div>
<nav class="bottom-nav"><a href="/" class="nav-item active"><i class="fas fa-home"></i><span>Home</span></a><a href="/explore" class="nav-item"><i class="fas fa-compass"></i><span>Explore</span></a><a href="/create" class="nav-item"><i class="fas fa-plus-square"></i><span>Create</span></a><a href="/activity" class="nav-item"><i class="far fa-heart"></i><span>Activity</span></a><a href="/profile" class="nav-item"><i class="far fa-user"></i><span>Profile</span></a></nav>
</div>
<script>window.__INITIAL_STATE__ = ${initialState};</script>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script src="/js/app.js" type="module"></script>
</body>
</html>`;

      const response = new Response(html, {
        headers: {
          'Content-Type': 'text/html; charset=utf-8',
          'Cache-Control': 'public, max-age=120, stale-while-revalidate=600',
          'Vary': 'Accept-Encoding',
          'Content-Security-Policy': "default-src 'self'; img-src * data: blob:; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://cdnjs.cloudflare.com; font-src https://fonts.gstatic.com https://cdnjs.cloudflare.com; script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; connect-src 'self' https://*.supabase.co wss://*.supabase.co; media-src * blob:",
          'X-Content-Type-Options': 'nosniff',
          'X-Frame-Options': 'DENY',
          'X-XSS-Protection': '1; mode=block',
          'Referrer-Policy': 'strict-origin-when-cross-origin'
        },
      });
      
      // Edge cache the HTML for 60 seconds
      await cache.put(htmlCacheKey, response.clone());
      
      return response;
      
    } catch (error) {
      return new Response(`<!DOCTYPE html><html lang="en"><head><meta charset="UTF-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><title>${SITE_NAME}</title><meta name="robots" content="noindex"><style>*{margin:0;padding:0}body{background:#000;color:#fff;text-align:center;padding:60px 20px;font-family:sans-serif;min-height:100vh;display:flex;align-items:center;justify-content:center;flex-direction:column}h1{font-size:48px;color:#F86D2A;margin-bottom:12px}p{font-size:16px;color:#A8A8A8;margin-bottom:24px}a{display:inline-block;background:#F86D2A;color:#fff;padding:14px 32px;border-radius:30px;text-decoration:none;font-weight:600}</style></head><body><h1>${SITE_NAME}</h1><p>We're experiencing high traffic. Please try again.</p><a href="/">Refresh Page</a></body></html>`, {
        headers: { 'Content-Type': 'text/html; charset=utf-8' },
        status: 500
      });
    }
  },
};

function esc(text) {
  if (!text) return '';
  return text.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}
