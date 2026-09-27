<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue';
import MapView from './components/MapView.vue';
import { cities } from './data/cities';
import { houses, topics } from './data/atlas';
const route = ref(location.hash.slice(1) || '/map');
const menuOpen = ref(false);
const query = ref('');
const region = ref('全部');
const mainHeading = ref<HTMLElement|null>(null);
const searchInput = ref<HTMLInputElement|null>(null);
const nav = [{path:'/map',name:'探索地图',en:'ATLAS'},{path:'/houses',name:'家族图鉴',en:'HOUSES'},{path:'/history',name:'历史与王朝',en:'CHRONICLES'},{path:'/world',name:'世界与社会',en:'WORLD'}];
const section = computed(()=>route.value.split('/')[1] || 'map');
const detail = computed(()=>route.value.split('/')[2]);
const house = computed(()=>houses.find(h=>h.id===detail.value));
const topic = computed(()=>topics.find(t=>t.id===detail.value));
const filteredHouses = computed(()=>houses.filter(h=>region.value==='全部'||h.region===region.value));
const results = computed(()=>{
 const q=query.value.trim().toLowerCase();if(!q)return [];
 return [...cities.map(c=>({name:c.name,kind:'地点',path:`/map/${c.id}`})),...houses.map(h=>({name:`${h.name} ${h.en}`,kind:'家族',path:`/houses/${h.id}`})),...topics.map(t=>({name:`${t.category} · ${t.title}`,kind:'专题',path:`/world/${t.id}`})),{name:'坦格利安 Targaryen · 征服初代族谱',kind:'历史',path:'/history'}].filter(r=>r.name.toLowerCase().includes(q));
});
function go(path:string){ if(location.hash===`#${path}`){menuOpen.value=false;return;} location.hash=path; }
async function onRoute(){route.value=location.hash.slice(1)||'/map';menuOpen.value=false;window.scrollTo(0,0);await nextTick();document.title=`${house.value&&section.value==='houses'?house.value.name+'家族':nav.find(n=>n.path===`/${section.value}`)?.name||'搜索'} · 维斯特洛图鉴`;mainHeading.value?.focus({preventScroll:true});if(section.value==='search')searchInput.value?.focus();}
onMounted(()=>{window.addEventListener('hashchange',onRoute);void onRoute();});
onBeforeUnmount(()=>window.removeEventListener('hashchange',onRoute));
function cityName(id:string){return cities.find(c=>c.id===id)?.name.split(' ')[0]||id;}
const person=ref('aegon');
const people=[{id:'aegon',name:'伊耿一世',role:'征服者 · 在位 1–37 AC',text:'与维桑尼亚、雷妮丝共同发动征服战争，以巨龙与联盟建立新的王权。伊尼斯一世与梅葛一世是他的两个儿子。'},{id:'visenya',name:'维桑尼亚',role:'伊耿的姐姐与王后',text:'骑乘瓦格哈尔的龙骑士，梅葛一世之母。她参与征服战争，与伊耿、雷妮丝共同构成王朝建立时期的核心。'},{id:'rhaenys',name:'雷妮丝',role:'伊耿的妹妹与王后',text:'骑乘米拉西斯的龙骑士，伊尼斯一世之母。她与兄姐共同参与征服战争。'},{id:'aenys',name:'伊尼斯一世',role:'伊耿与雷妮丝之子 · 在位 37–42 AC',text:'伊耿之后的第二位坦格利安国王。他的统治让王朝面临继承、信仰与王权之间的挑战。'},{id:'maegor',name:'梅葛一世',role:'伊耿与维桑尼亚之子 · 在位 42–48 AC',text:'伊尼斯的同父异母弟弟，伊耿之后的第三位国王。这一支关系说明：血缘世代与王位继承顺序并不总是同一回事。'}];
const selectedPerson=computed(()=>people.find(p=>p.id===person.value)!);
</script>

<template>
  <div class="atlas-shell">
    <a class="skip-link" href="#content" @click.prevent="mainHeading?.focus()">跳到正文</a>
    <header class="site-header">
      <a class="brand" href="#/map" aria-label="维斯特洛图鉴首页"><span class="brand-mark">W</span><span><strong>维斯特洛图鉴</strong><small>THE WESTEROS ATLAS</small></span></a>
      <nav class="desktop-nav" aria-label="主导航"><a v-for="item in nav" :key="item.path" :href="`#${item.path}`" :aria-current="route.startsWith(item.path)?'page':undefined">{{item.name}}</a></nav>
      <div class="header-actions"><a class="search-toggle" href="#/search" aria-label="搜索图鉴"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" aria-hidden="true"><circle cx="10.5" cy="10.5" r="6.5"/><path d="m16 16 5 5"/></svg><span>搜索</span></a><button class="menu-toggle" :aria-expanded="menuOpen" aria-controls="mobile-nav" @click="menuOpen=!menuOpen">{{menuOpen?'关闭':'目录'}} <span aria-hidden="true">{{menuOpen?'×':'☰'}}</span></button></div>
    </header>
    <nav v-if="menuOpen" id="mobile-nav" class="mobile-nav" aria-label="手机导航" @keydown.esc="menuOpen=false"><a v-for="(item,i) in nav" :key="item.path" :href="`#${item.path}`" @click="menuOpen=false"><small>0{{i+1}} / {{item.en}}</small>{{item.name}} <span>↗</span></a></nav>
    <div id="content" ref="mainHeading" tabindex="-1" class="content-root">
      <MapView v-if="section==='map'" :city-id="detail" @navigate="go" />
      <main v-else class="reading-page">
        <template v-if="section==='houses' && !detail">
          <header class="page-intro"><p class="eyebrow">VOLUME II / THE GREAT HOUSES</p><h1>纹章之下，<br>皆有来历。</h1><p class="intro-copy">从北境的冰原狼到河间地的银鳟，<br class="desktop-only">沿着领地、血缘与誓言，认识维斯特洛的家族。</p><span class="edition-note">首辑收录 {{houses.length}} 个家族 · 原著背景导读</span></header>
          <div class="section-bar"><h2>家族索引 <small>THE HOUSES</small></h2><div class="filters" aria-label="按地区筛选"><button v-for="r in ['全部','北境','西境','河间地','河湾地']" :key="r" :aria-pressed="region===r" @click="region=r">{{r}}</button></div></div>
          <div class="house-grid"><a v-for="h in filteredHouses" :key="h.id" :href="`#/houses/${h.id}`" class="house-card"><span class="house-region">{{h.region}} / {{h.seat}}</span><img :src="h.sigil" :alt="`${h.name}家徽`" width="90" height="105" loading="lazy"><small>HOUSE {{h.en}}</small><h2>{{h.name}}家族</h2><p>{{h.summary}}</p><span class="card-link">阅读家族档案 <b>↗</b></span></a></div>
        </template>
        <template v-else-if="section==='houses' && house">
          <a class="back-link" href="#/houses">← 家族索引</a><div class="house-detail"><aside class="heraldry"><img :src="house.sigil" :alt="`${house.name}家徽`"><p>HOUSE {{house.en}}</p><h2>{{house.words}}</h2><small>{{house.englishWords}}</small><dl><dt>领地</dt><dd>{{house.region}}</dd><dt>居城</dt><dd>{{house.seat}}</dd></dl></aside><article class="article-body"><p class="eyebrow">家族档案 / {{house.region}}</p><h1>{{house.name}}家族</h1><p class="article-lead">{{house.summary}}</p><h2>家族渊源</h2><p v-for="p in house.paragraphs" :key="p">{{p}}</p><a class="primary-link" :href="`#/map/${house.city}`">在地图中探访{{house.seat}} <span>↗</span></a><template v-if="house.related.length"><h2>关联家族</h2><p class="muted">沿着封臣关系与家族联姻继续阅读。</p><div class="related-links"><a v-for="id in house.related" :key="id" :href="`#/houses/${id}`">{{houses.find(h=>h.id===id)?.name}}家族 ↗</a></div></template><a class="source-link" :href="house.source" target="_blank" rel="noopener noreferrer">参考资料 · A Wiki of Ice and Fire ↗</a></article></div>
        </template>
        <template v-else-if="section==='history'">
          <header class="page-intro"><p class="eyebrow">VOLUME III / FIRE & BLOOD</p><h1>王朝始于龙焰，<br>延续于血脉。</h1><p class="intro-copy">坦格利安王朝 · 从征服初代开始阅读。<br>本页展示初代两支血缘及前三位君主，并非完整族谱。</p></header>
          <div class="section-bar"><h2>征服者与继承者 <small>THE FIRST GENERATIONS</small></h2><span class="edition-note">AC = 伊耿征服后纪年 · 含王朝历史</span></div>
          <section class="dynasty-layout" aria-label="坦格利安初代家族关系"><div class="family-tree"><div class="tree-founder"><button @click="person='aegon'" :aria-pressed="person==='aegon'"><small>征服者</small><strong>伊耿一世</strong><span>两段婚姻 · 两支血脉</span></button></div><div class="tree-branches"><div v-for="pair in [['visenya','maegor'],['rhaenys','aenys']]" :key="pair[0]" class="tree-branch"><span class="relationship">与伊耿成婚</span><button v-for="(id,i) in pair" :key="id" @click="person=id" :aria-pressed="person===id"><small>{{i?'儿子':'姐妹与王后'}}</small><strong>{{people.find(p=>p.id===id)?.name}}</strong></button></div></div><p class="muted">选择人物，查看关系说明。连线表示婚姻分支与母子关系。</p></div><article class="person-detail" aria-live="polite"><p class="eyebrow">PERSON / 人物</p><h2>{{selectedPerson.name}}</h2><p class="person-role">{{selectedPerson.role}}</p><p>{{selectedPerson.text}}</p></article></section>
          <section class="reign-list"><h2>王位继承顺序</h2><a v-for="(p,i) in [people[0],people[3],people[4]]" :key="p!.id" href="#/history" @click.prevent="person=p!.id"><span>0{{i+1}}</span><strong>{{p!.name}}</strong><small>{{p!.role.split('·').at(-1)}}</small></a></section><a class="source-link" href="https://awoiaf.westeros.org/index.php/House_Targaryen" target="_blank" rel="noopener noreferrer">参考资料 · House Targaryen ↗</a>
        </template>
        <template v-else-if="section==='world' && !detail">
          <header class="page-intro"><p class="eyebrow">VOLUME IV / LIFE BEYOND THE THRONE</p><h1>王座之外，<br>世界如何运转。</h1><p class="intro-copy">信仰塑造誓言，季节决定收成。<br>从四个切面，读懂地图上的日常与秩序。</p></header>
          <div class="topic-list"><a v-for="(t,i) in topics" :key="t.id" :href="`#/world/${t.id}`"><span class="topic-number">0{{i+1}}</span><div><p class="eyebrow">{{t.category}}</p><h2>{{t.title}}</h2><p>{{t.subtitle}}</p></div><span class="topic-arrow">↗</span></a></div>
        </template>
        <template v-else-if="section==='world' && topic">
          <a class="back-link" href="#/world">← 世界与社会</a><article class="topic-article article-body"><p class="eyebrow">专题导读 / {{topic.category}}</p><h1>{{topic.title}}</h1><p class="article-lead">{{topic.subtitle}}</p><section v-for="s in topic.sections" :key="s.title"><h2>{{s.title}}</h2><p>{{s.text}}</p></section><h2>回到地图</h2><div class="related-links"><a v-for="id in topic.cities" :key="id" :href="`#/map/${id}`">{{cityName(id)}} ↗</a></div><a class="source-link" :href="`https://awoiaf.westeros.org/index.php/${topic.source}`" target="_blank" rel="noopener noreferrer">参考资料 · {{topic.source}} ↗</a></article>
        </template>
        <template v-else-if="section==='search'"><header class="page-intro"><p class="eyebrow">INDEX / 全站索引</p><h1>你想从哪里开始？</h1><p class="intro-copy">寻找地点、家族，或一个关于世界的问题。</p></header><label class="search-label" for="atlas-search">搜索图鉴</label><input id="atlas-search" ref="searchInput" v-model="query" type="search" class="search-input" placeholder="试试：奔流城、Stark、宗教…"><p class="muted" role="status">{{query.trim()?`找到 ${results.length} 条结果`:'输入中文或英文名称，探索已收录的内容。'}}</p><div class="search-results"><a v-for="r in results" :key="r.path" :href="`#${r.path}`"><small>{{r.kind}}</small><strong>{{r.name}}</strong><span>↗</span></a></div><p v-if="query.trim()&&!results.length">暂未收录这个词，试试家族姓氏或城堡名称。</p></template>
        <template v-else><h1>这卷书尚未收录此页。</h1><a class="primary-link" href="#/map">返回地图 →</a></template>
        <footer class="site-footer"><span>WESTEROS / 一部正在生长的世界图鉴</span><span>以原著背景为主 · 同人非官方项目</span></footer>
      </main>
    </div>
  </div>
</template>
