---
ctime: "2026-01-10T14:38:05+08:00"
mtime: "2026-01-10T14:38:05+08:00"
---

# script-site-moc-builder

```js

const tableImageWidth = 50;

function parseWikiLink(wikilink) {
	if (!wikilink || typeof wikilink !== 'string') return null;
	const m = /^\[\[([^\|\]]+)(?:\|.*)?\]\]$/.exec(wikilink.trim());
	return m ? m[1] : null;
}

function getImageElem(wikilink, width=200) {
	const linktext = parseWikiLink(wikilink);
	if (!linktext) return '';
	const file = app.metadataCache.getFirstLinkpathDest(linktext);
	if (!file || !file.path) return '';
	return `<img src="${encodeURI(file.path)}" width="${width}">`;
}

function slugFromBasename(basename, prefix) {
	const re = new RegExp('^' + prefix + '(.*)');
	const m = re.exec(basename);
	return m ? m[1] : basename;
}

function safeFrontmatter(file) {
	const fileCache = app.metadataCache.getFileCache(file) || {};
	return fileCache.frontmatter || {};
}

function safeArray(value) {
	if (!value) {return [];}
	if (Array.isArray(value)) {return value;}
	return [value];
}

function getCategoryArrStr(itemFile, categoryFiles) {
	const fm = safeFrontmatter(itemFile);
	const categories = safeArray(fm.categories).map(l => {
		const pt = parseWikiLink(l) || l;
		const i = categoryFiles.findIndex(f=>f.basename===pt);
		const display = pt.replace(/^site-category-/, '');
		return `[${display}](#${i+1}-${display})`;
	}).join(', ');
	return categories;
}

function getItemArrStr(categoryFile, m) {
	const fm = safeFrontmatter(categoryFile);
	const items = safeArray(fm.subpages).map((l,j)=>{
		const pt = parseWikiLink(l) || l;
		const n = j + 1;
		const slug = slugFromBasename(pt, 'site-item-');
		return `[${slug}](#${m}-${n}-${slug})`;
	}).join(', ');
	return items;
}

function getSiteItemSection(itemFile, m, n, categoryFiles) {
	const slug = slugFromBasename(itemFile.basename, 'site-item-');
	const title = `${m}-${n}-${slug}`;
	const fm = safeFrontmatter(itemFile);

	const headerMarker = m !== 0 ? '####' : '###';
	const displayTitle = fm.title || slug;
	const url = fm.url || '#';
	const description = fm.description || '';
	const categories = getCategoryArrStr(itemFile, categoryFiles)
	const icon = fm.icon ? getImageElem(fm.icon) : '';

	return `${headerMarker} ${title}\n\n> see: [${displayTitle}](${url})\n\n${description}\n\nCategories: ${categories}\n\n${icon}`;
}

function getSiteCategorySection(file, m, categoryFiles) {
	const slug = slugFromBasename(file.basename, 'site-category-');
	const title = `${m}-${slug}`;
	const fm = safeFrontmatter(file);

	const subpages = safeArray(fm.subpages);

	const ol = `> [!Note]\n> \n> #### [site-items](#site-items)/[category](#category)/**${slug}**/\n> \n> | \\# | [Site-Items](#site-items) | [Category](#category) | Icon | Description |\n> | --- | --- | --- | --- | --- |\n`+subpages.map((link, j) => {
		const n = j + 1;
		const linkTarget = parseWikiLink(link) || link;
		const file = app.metadataCache.getFirstLinkpathDest(linkTarget)
		const slug = slugFromBasename(linkTarget, 'site-item-');
		const fm = safeFrontmatter(file);
		const icon = getImageElem(fm.icon,tableImageWidth);
		const target = `#${m}-${n}-${slug}`;
		const description = fm.description;
		return `> | ${n} | [${slug}](${target}) | ${getCategoryArrStr(file, categoryFiles)} | [${icon}](${target}) | ${description} |`;
	}).join('\n');

	const relatedSiteItems = subpages.map(link => {
		const linktext = parseWikiLink(link) || link;
		return app.metadataCache.getFirstLinkpathDest(linktext);
	}).filter(Boolean);

	const itemSections = relatedSiteItems.map((f, j) => getSiteItemSection(f, m, j + 1, categoryFiles)).join('\n\n');

	return `### ${title}\n\n${ol}\n\n${itemSections}`;
}

try {
	const mdfiles = app.vault.getMarkdownFiles();
	const siteitems = mdfiles.filter(f => f.path.startsWith('site-item/')).sort((a, b) => a.basename.localeCompare(b.basename));
	const sitecategories = mdfiles.filter(f => f.path.startsWith('site-category/')).sort((a, b) => a.basename.localeCompare(b.basename));

	const parts = [];

	// Category index
	parts.push('## category\n');
	parts.push("> [!Note]\n> \n> #### [site-items](#site-items)/category/\n> \n> | \\# | [Category](#category) | [Site-Items](#site-items) | Icon |\n> | --- | --- | --- | --- |\n"+sitecategories.map((f, i) => {
		const m = i + 1;
		const slug = slugFromBasename(f.basename, 'site-category-');
		const target = `#${m}-${slug}`;
		
		const iconArrStr = safeFrontmatter(f).subpages.map((l,j)=>{
			const n = j + 1;
			const itemFile = app.metadataCache.getFirstLinkpathDest(parseWikiLink(l) || l);
			const linkTarget = parseWikiLink(l) || l;
			const slug = slugFromBasename(linkTarget, 'site-item-');
			const icon = getImageElem(safeFrontmatter(itemFile).icon,tableImageWidth);
			const targetToItem = `#${m}-${n}-${slug}`;
			return `[${icon}](${targetToItem})`;
		}).join("");
		
		return `> | ${m} | [${slug}](${target}) | ${getItemArrStr(f,m)} | ${iconArrStr} |`;
	}).join('\n'));
	parts.push('\n');
	parts.push(sitecategories.map((f, i) => getSiteCategorySection(f, i + 1, sitecategories)).join('\n\n'));

	// Site items index and sections
	parts.push('\n## site-items\n\n');
	parts.push("> [!Note]\n> \n> #### site-items/\n> \n> | \\# | [Site-Items](#site-items) | [Category](#category) | Icon | Description |\n> | --- | --- | --- | --- | --- |\n"+siteitems.map((f, j) => {
		const n = j + 1;
		const slug = slugFromBasename(f.basename, 'site-item-');
		const target = `#0-${n}-${slug}`;
		const fm = safeFrontmatter(f);
		const icon = getImageElem(fm.icon,tableImageWidth);
		const description = fm.description;
		return `> | ${n} | [${slug}](${target}) | ${getCategoryArrStr(f, sitecategories)} | [${icon}](${target}) | ${description} |`;
	}).join('\n'));
	parts.push('\n\n');
	parts.push(siteitems.map((f, j) => getSiteItemSection(f, 0, j + 1, sitecategories)).join('\n\n'));

	console.log(parts.join('\n')+'\n');
} catch (err) {
	console.error('Error building site MOC:', err);
}

```
