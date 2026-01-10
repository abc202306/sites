---
ctime: "2026-01-10T14:38:05+08:00"
mtime: "2026-01-10T14:38:05+08:00"
---

# script-site-moc-builder

```js
function getImageElem(wikilink) {
	const linktext = /^\[\[(.*)(\\?\|.*)?\]\]$/.exec(wikilink)[1];
	const file = app.metadataCache.getFirstLinkpathDest(linktext);
	return `<img src="${file.path}" width=200>`;
}
function getSiteItemSection(file, m, n) {
	const title = `${m}-${n}-`+/^site-item-(.*)/.exec(file.basename)[1];
	const fileCache = app.metadataCache.getFileCache(file);
	const fm = fileCache.frontmatter;
	
	let headerMarker = "###";
	if (m !== 0) {;
		headerMarker = "####";
	}
	
	return `${headerMarker} ${title}\n\n> see: [${fm['title']}](${fm['url']})\n\n${fm['description']}\n\ncategories: ${fm['categories'].map(l=>/^\[\[(site-category-)?(?<value>.*?)(\\?\|.*)?\]\]$/.exec(l).groups['value']).join(", ")}\n\n${getImageElem(fm['icon'])}`
}
function getSiteCategorySection(file,m) {
	const title = m+"-"+/^site-category-(.*)/.exec(file.basename)[1];
	const fileCache = app.metadataCache.getFileCache(file);
	const fm = fileCache.frontmatter;
	
	const ol = fm['subpages'].map((link,j)=>{
		const n = j+1;
		const mangaitemsectionid = /^\[\[site-item-([^\|]*)/.exec(link)[1];
		return `1. [${m}-${n}-${mangaitemsectionid}](#${m}-${n}-${mangaitemsectionid})`
	}).join("\n");
	
	const relatedSiteItems = fm['subpages'].map(link=>{
		const linktext = /^\[\[([^\|]*)/.exec(link)[1];
		return app.metadataCache.getFirstLinkpathDest(linktext);
	})
	
	return `### ${title}\n\n${ol}\n\n${relatedSiteItems.map((f,j)=>getSiteItemSection(f,m,j+1)).join("\n\n")}`;
}
const mdfiles = app.vault.getMarkdownFiles();
const siteitems = mdfiles.filter(f=>f.path.startsWith("site-item/")).sort();
const sitecategories = mdfiles.filter(f=>f.path.startsWith("site-category/")).sort();
console.log(`## category\n\n${sitecategories.map((f,i)=>{const m = i + 1; const title = m+"-"+/^site-category-(.*)/.exec(f.basename)[1]; return `1. [${title}](#${title})`}).join("\n")}\n\n${sitecategories.map((f,i)=>getSiteCategorySection(f,i+1)).join("\n\n")}\n\n## site-items\n\n${siteitems.map((f,j)=>{const n = j + 1; const title = `0-${n}-`+/^site-item-(.*)/.exec(f.basename)[1]; return `1. [${title}](#${title})`;}).join("\n")}\n\n${siteitems.map((f,j)=>getSiteItemSection(f,0,j+1)).join("\n\n")}\n`)

```
