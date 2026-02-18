+++
title = "Cite"
date = 2026-02-18
+++

**MLA**

<div class="citation" onclick="copyText(this)">
Myers, Michael J. "A Study: Maupassant in Translation." 2026, https://myersm0.github.io/maupassant-study/.
</div>

**APA**

<div class="citation" onclick="copyText(this)">
Myers, M. J. (2026). <em>A Study: Maupassant in translation</em>. https://myersm0.github.io/maupassant-study/
</div>

**BibTeX**

<div class="citation" onclick="copyText(this)">
<pre>@misc{myers-maupassant-study,
  author = {Myers, Michael J.},
  title = {A Study: Maupassant in Translation},
  year = {2026},
  url = {https://myersm0.github.io/maupassant-study/}
}</pre>
</div>

<small>Click any citation to copy.</small>

<script>
function copyText(el) {
	const text = el.innerText;
	navigator.clipboard.writeText(text);
	el.classList.add('copied');
	setTimeout(() => el.classList.remove('copied'), 1000);
}
</script>
