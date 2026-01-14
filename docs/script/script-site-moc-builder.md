---
ctime: "2026-01-10T14:38:05+08:00"
mtime: "2026-01-10T14:38:05+08:00"
---

# script-site-moc-builder

```js

function escapePipe(str) {
	return str.replace(/(?<!\\)\|/g,"\\|");
}

function getDescriptionInTable(fm) {
	return `[${escapePipe(fm.title)}](${fm.url})<br><br>${fm.description}`;
}

class WikiParser {
    constructor(metadataCache, options = {}) {
        this.metadataCache = metadataCache || null;
        this.options = Object.assign({ defaultImageWidth: 200 }, options);
    }

    parseWikiLink(wikilink) {
        if (!wikilink || typeof wikilink !== 'string') return null;
        const m = /^\[\[([^\|\]]+)(?:\|.*)?\]\]$/.exec(wikilink.trim());
        return m ? m[1] : null;
    }

    getFile(linktext) {
        return this.metadataCache ? this.metadataCache.getFirstLinkpathDest(linktext) : null;
    }

    getImageElem(wikilink, width = this.options.defaultImageWidth) {
        const linktext = this.parseWikiLink(wikilink);
        if (!linktext) return '';
        const file = this.getFile(linktext);
        if (!file || !file.path) return '';
        return `<img src="${encodeURI(file.path)}" width="${width}">`;
    }

    slugFromBasename(basename, prefix) {
        if (typeof basename !== 'string' || typeof prefix !== 'string') return basename;
        return basename.startsWith(prefix) ? basename.slice(prefix.length) : basename;
    }
}

class FrontmatterHelper {
    constructor(metadataCache) {
        this.metadataCache = metadataCache || null;
    }

    safeFrontmatter(file) {
        const fileCache = this.metadataCache ? (this.metadataCache.getFileCache(file) || {}) : {};
        return fileCache.frontmatter || {};
    }

    safeArray(value) {
        if (!value) return [];
        if (Array.isArray(value)) return value;
        if (typeof value === 'string') return [value];
        return [value];
    }
}

class SiteRenderer {
    constructor(parser, fmHelper, options = {}) {
        this.parser = parser;
        this.fm = fmHelper;
        this.options = Object.assign({ tableImageWidth: 50 }, options);
    }

    getCategoryArrStr(itemFile, categoryFiles, emphasizedText) {
        const fm = this.fm.safeFrontmatter(itemFile);
        return this.fm.safeArray(fm.categories).map(l => {
            const pt = this.parser.parseWikiLink(l) || l;
            const i = categoryFiles.findIndex(f => f.basename === pt);
            const displayItem = pt.replace(/^site-category-/, '');
            const display = displayItem === emphasizedText ? `**${displayItem}**` : displayItem;
            return `[${display}](#${i + 1}-${displayItem})`;
        }).join(', ');
    }

    getItemArrStr(categoryFile, m) {
        const fm = this.fm.safeFrontmatter(categoryFile);
        return this.fm.safeArray(fm.subpages).map((l, j) => {
            const pt = this.parser.parseWikiLink(l) || l;
            const n = j + 1;
            const slug = this.parser.slugFromBasename(pt, 'site-item-');
            return `[${slug}](#${m}-${n}-${slug})`;
        }).join(', ');
    }

    getSiteItemSection(itemFile, m, n, categoryFiles, emphasizedText) {
        const slug = this.parser.slugFromBasename(itemFile.basename, 'site-item-');
        const title = `${m}-${n}-${slug}`;
        const fm = this.fm.safeFrontmatter(itemFile);
        const headerMarker = m !== 0 ? '####' : '###';
        const displayTitle = fm.title || slug;
        const url = fm.url || '#';
        const description = fm.description || '';
        const categories = this.getCategoryArrStr(itemFile, categoryFiles, emphasizedText);
        const icon = fm.icon ? this.parser.getImageElem(fm.icon) : '';

        return `${headerMarker} ${title}\n\n> see: [${displayTitle}](${url})\n\n${description}\n\n${icon}\n\n> [!Note]\n> \n> | | |\n> | --- | --- |\n> | [Category](#category) |  ${categories} |\n> | [Type](#type) | [site-items](#site-items) |`;
    }

    getSiteCategorySection(file, m, categoryFiles) {
        const slugCategory = this.parser.slugFromBasename(file.basename, 'site-category-');
        const title = `${m}-${slugCategory}`;
        const fm = this.fm.safeFrontmatter(file);
        const subpages = this.fm.safeArray(fm.subpages);

        const olHeader = `> [!Note]\n> \n> #### [type](#type)/[category](#category)/**${slugCategory}**/\n> \n> | \\# | [**Site-Items**](#site-items) | [**Category**](#category) | Icon | Description |\n> | --- | --- | --- | --- | --- |\n`;

        const olRows = subpages.map((link, j) => {
            const n = j + 1;
            const linkTarget = this.parser.parseWikiLink(link) || link;
            const file = this.parser.getFile(linkTarget);
            const slugItem = this.parser.slugFromBasename(linkTarget, 'site-item-');
            const fm = this.fm.safeFrontmatter(file);
            const icon = this.parser.getImageElem(fm.icon, this.options.tableImageWidth);
            const target = `#${m}-${n}-${slugItem}`;
            const description = getDescriptionInTable(fm);
            return `> | ${n} | [${slugItem}](${target}) | ${this.getCategoryArrStr(file, categoryFiles, slugCategory)} | [${icon}](${target}) | ${description} |`;
        }).join('\n');

        const relatedSiteItems = subpages.map(link => {
            const linktext = this.parser.parseWikiLink(link) || link;
            return this.parser.getFile(linktext);
        }).filter(Boolean);

        const itemSections = relatedSiteItems.map((f, j) => this.getSiteItemSection(f, m, j + 1, categoryFiles, slugCategory)).join('\n\n');

        return `### ${title}\n\n${olHeader}${olRows}\n\n${itemSections}`;
    }
}

class MOCBuilder {
    constructor(metadataCache, vault, options = {}) {
        this.metadataCache = metadataCache || null;
        this.vault = vault || null;
        this.options = Object.assign({ tableImageWidth: 50, defaultImageWidth: 200 }, options);
        this.parser = new WikiParser(this.metadataCache, { defaultImageWidth: this.options.defaultImageWidth });
        this.fm = new FrontmatterHelper(this.metadataCache);
        this.renderer = new SiteRenderer(this.parser, this.fm, { tableImageWidth: this.options.tableImageWidth });
    }

    // gather markdown site files from vault and sort them
    _getSiteFiles() {
        const mdfiles = this.vault ? this.vault.getMarkdownFiles() : [];
        const siteitems = mdfiles
            .filter(f => f.path.startsWith('site-item/'))
            .sort((a, b) => a.basename.localeCompare(b.basename));
        const sitecategories = mdfiles
            .filter(f => f.path.startsWith('site-category/'))
            .sort((a, b) => a.basename.localeCompare(b.basename));
        return { siteitems, sitecategories };
    }

    // build the category index table (first section)
    _buildCategoryIndex(sitecategories) {
        return (
            "> [!Note]\n> \n> ### [type](#type)/category/\n> \n> | \\\# | **Category** | [**Site-Items**](#site-items) | Icon |\n> | --- | --- | --- | --- |\n" +
            sitecategories
                .map((f, i) => {
                    const m = i + 1;
                    const slug = this.parser.slugFromBasename(f.basename, 'site-category-');
                    const target = `#${m}-${slug}`;

                    const iconArrStr = this.fm
                        .safeArray(this.fm.safeFrontmatter(f).subpages)
                        .map((l, j) => {
                            const n = j + 1;
                            const linkTarget = this.parser.parseWikiLink(l) || l;
                            const itemFile = this.parser.getFile(linkTarget);
                            const slug = this.parser.slugFromBasename(linkTarget, 'site-item-');
                            const icon = this.parser.getImageElem(this.fm.safeFrontmatter(itemFile).icon, this.options.tableImageWidth);
                            const targetToItem = `#${m}-${n}-${slug}`;
                            return `[${icon}](${targetToItem})`;
                        })
                        .join('');

                    return `> | ${m} | [${slug}](${target}) | ${this.renderer.getItemArrStr(f, m)} | ${iconArrStr} |`;
                })
                .join('\n')
        );
    }

    _buildCategorySections(sitecategories) {
        return sitecategories.map((f, i) => this.renderer.getSiteCategorySection(f, i + 1, sitecategories)).join('\n\n');
    }

    _buildSiteItemsIndex(siteitems, sitecategories) {
        return (
            "> [!Note]\n> \n> ### [type](#type)/site-items/\n> \n> | \\\# | **Site-Items** | [**Category**](#category) | Icon | Description |\n> | --- | --- | --- | --- | --- |\n" +
            siteitems
                .map((f, j) => {
                    const n = j + 1;
                    const slug = this.parser.slugFromBasename(f.basename, 'site-item-');
                    const target = `#0-${n}-${slug}`;
                    const fm = this.fm.safeFrontmatter(f);
                    const icon = this.parser.getImageElem(fm.icon, this.options.tableImageWidth);
                    const description = getDescriptionInTable(fm);
                    return `> | ${n} | [${slug}](${target}) | ${this.renderer.getCategoryArrStr(f, sitecategories)} | [${icon}](${target}) | ${description} |`;
                })
                .join('\n')
        );
    }

    _buildSiteItemsSections(siteitems, sitecategories) {
        return siteitems.map((f, j) => this.renderer.getSiteItemSection(f, 0, j + 1, sitecategories)).join('\n\n');
    }

    build() {
        const { siteitems, sitecategories } = this._getSiteFiles();

        const parts = [];
        parts.push('## type\n\n> [!Note]\n> \n> ### type\n> \n> 1. [category](#category)\n> 1. [site-items](#site-items)');
        parts.push('\n## category\n');
        parts.push(this._buildCategoryIndex(sitecategories));
        parts.push('\n');
        parts.push(this._buildCategorySections(sitecategories));

        parts.push('\n## site-items\n\n');
        parts.push(this._buildSiteItemsIndex(siteitems, sitecategories));
        parts.push('\n\n');
        parts.push(this._buildSiteItemsSections(siteitems, sitecategories));
        

        return parts.join('\n') + '\n';
    }
}

// bootstrap and run
try {
    const metadataCache = (typeof app !== 'undefined' && app.metadataCache) ? app.metadataCache : null;
    const vault = (typeof app !== 'undefined' && app.vault) ? app.vault : null;
    const builder = new MOCBuilder(metadataCache, vault);
    const output = builder.build();
    console.log(output);
} catch (err) {
    console.error('Error building site MOC:', err);
}

```
