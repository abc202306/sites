---
ctime: "2026-01-10T14:38:05+08:00"
mtime: "2026-01-10T14:38:05+08:00"
---

# script-site-moc-builder

```js
function parseWikiLink(wikilink) {
	if (!wikilink || typeof wikilink !== 'string') return null;
	const m = /^\[\[([^\|\]]+)(?:\|.*)?\]\]$/.exec(wikilink.trim());
	return m ? m[1] : null;
}

function getImageElem(wikilink) {
	const linktext = parseWikiLink(wikilink);
	if (!linktext) return '';
	const file = app.metadataCache.getFirstLinkpathDest(linktext);
	if (!file || !file.path) return '';
	return `<img src="${encodeURI(file.path)}" width="200">`;
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

function getSiteItemSection(file, m, n) {
	const slug = slugFromBasename(file.basename, 'site-item-');
	const title = `${m}-${n}-${slug}`;
	const fm = safeFrontmatter(file);

	const headerMarker = m !== 0 ? '####' : '###';
	const displayTitle = fm.title || slug;
	const url = fm.url || '#';
	const description = fm.description || '';
	const categoriesArr = Array.isArray(fm.categories) ? fm.categories : [];
	const categories = categoriesArr.map(l => {
		const pt = parseWikiLink(l) || l;
		return pt.replace(/^site-category-/, '');
	}).join(', ');
	const icon = fm.icon ? getImageElem(fm.icon) : '';

	return `${headerMarker} ${title}\n\n> see: [${displayTitle}](${url})\n\n${description}\n\nCategories: ${categories}\n\n${icon}`;
}

function getSiteCategorySection(file, m) {
	const slug = slugFromBasename(file.basename, 'site-category-');
	const title = `${m}-${slug}`;
	const fm = safeFrontmatter(file);

	const subpages = Array.isArray(fm.subpages) ? fm.subpages : [];

	const ol = subpages.map((link, j) => {
		const n = j + 1;
		const linkTarget = parseWikiLink(link) || link;
		const mangaitemsectionid = slugFromBasename(linkTarget, 'site-item-');
		return `1. [${m}-${n}-${mangaitemsectionid}](#${m}-${n}-${mangaitemsectionid})`;
	}).join('\n');

	const relatedSiteItems = subpages.map(link => {
		const linktext = parseWikiLink(link) || link;
		return app.metadataCache.getFirstLinkpathDest(linktext);
	}).filter(Boolean);

	const itemSections = relatedSiteItems.map((f, j) => getSiteItemSection(f, m, j + 1)).join('\n\n');

	return `### ${title}\n\n${ol}\n\n${itemSections}`;
}

try {
	const mdfiles = app.vault.getMarkdownFiles();
	const siteitems = mdfiles.filter(f => f.path.startsWith('site-item/')).sort((a, b) => a.basename.localeCompare(b.basename));
	const sitecategories = mdfiles.filter(f => f.path.startsWith('site-category/')).sort((a, b) => a.basename.localeCompare(b.basename));

	const parts = [];

	// Category index
	parts.push('## category\n');
	parts.push(sitecategories.map((f, i) => {
		const m = i + 1;
		const t = `${m}-${slugFromBasename(f.basename, 'site-category-')}`;
		return `1. [${t}](#${t})`;
	}).join('\n'));
	parts.push('\n');
	parts.push(sitecategories.map((f, i) => getSiteCategorySection(f, i + 1)).join('\n\n'));

	// Site items index and sections
	parts.push('\n## site-items\n\n');
	parts.push(siteitems.map((f, j) => {
		const n = j + 1;
		const title = `0-${n}-${slugFromBasename(f.basename, 'site-item-')}`;
		return `1. [${title}](#${title})`;
	}).join('\n'));
	parts.push('\n\n');
	parts.push(siteitems.map((f, j) => getSiteItemSection(f, 0, j + 1)).join('\n\n'));

	console.log(parts.join('\n'));
} catch (err) {
	console.error('Error building site MOC:', err);
}

```
