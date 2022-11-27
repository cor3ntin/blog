---
title: "Augeas"
date: 2022-07-16T10:21:04+02:00
draft: false
---

<p style="display:none">
C++ is not done yet.
</p>
<!--more-->

<script>

function shuffle(array) {
  for (let i = array.length - 1; i > 0; i--) {
    let j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array
}

function shuffle_items() {
    var list = document.getElementById("list");
    var nodes = list.children;
    nodes = Array.prototype.slice.call(nodes);
    nodes = shuffle(nodes);
    list.append(...nodes);
}

window.onload = (event) => {
 shuffle_items();
};

</script>

<style>
.core {
    color: #008E89;
}
.library {
    color: #084594;
}
.async {
    color: #2C3639;
}
.text {
    color: #A149FA;
}
.perf {
    color: #EB4747;
}

.unicorn > span {
   background-image: linear-gradient(to left, violet, indigo, blue, green, #d2d20f, #eb9c0b, red);
   -webkit-background-clip: text;
   -webkit-text-fill-color: transparent;
}

.medium {
  font-size: 1.175em
}

.big {
  font-size: 1.3em
}


</style>

<ul id="list">
<li class="core big">Reflection on language constructs.</li>
<li class="core big">Programmatically inject code.</li>
<li class="core medium">Scoped and typesafe expressions expansion (hygienic macros).</li>
<li class="library medium">Starting and controlling child processes.</li>
<li class="async big">A framework for asynchrony and heterogeneous programming (<abbr title="Senders/Receivers">S/R</abbr>).</li>
<li class="async">A standard executor to make asynchrony available out of the box.</li>
<li class="unicorn"><span>Fixed-sized non-allocating coroutines and deferred layouts</span>.</li>
<li class="async">More ergonomic coroutines APIs.</li>
<li class="perf">Guaranteed tail calls.</li>
<li class="library">Mapping of system memory and files.</li>
<li class="library">More user-friendly Pseudo Random Number Generators.</li>
<li class="async">Library support for SIMD.</li>
<li class="async">SIMD algorithms.</li>
<li class="async">Parallel algorithms for ranges.</li>
<li class="async big">Asynchronous Networking facilities.</li>
<li class="async">Parallel execution and executors integration.</li>
<li class="perf">Small buffer optimization for <code>std::vector</code>.</li>
<li class="library">Open addressing hash map and set.</li>
<li class="text medium">Unicode algorithms.</li>
<li class="text">Unicode properties.</li>
<li class="text">Tailored Unicode algorithms.</li>
<li class="text medium">Unicode/BCP 47 locales.</li>
<li class="text">Collation.</li>
<li class="text">cldr-compliant localized date formatting.</li>
<li class="text">cldr-compliant localized number formatting.</li>
<li class="text">User-friendly encoding conversions.</li>
<li class="library">A generic facility to get the extent of an array.</li>
<li class="library"><code>constexpr</code> formatting.</li>
<li class="core">Expressions in <code>static_assert</code> messages.</li>
<li class="core">Compile time assertions.</li>
<li class="core medium">Contracts (assertions).</li>
<li class="perf">Non-throwing default allocator.</li>
<li class="core">Terser lambda syntax.</li>
<li class="core">Language construct for <code>std::forward</code>.</li>
<li class="core">Pack indexing.</li>
<li class="core">Pack as members.</li>
<li class="core">Non-trailing packs deduction.</li>
<li class="core">Universal template parameters.</li>
<li class="core medium">First-class support for Customization Points Objects, Niebloids and Overload Sets.</li>
<li class="library">Trees and ropes types.</li>
<li class="library">Improving the <code>inserter</code> interface for maps and sets.</li>
<li class="core">Concepts and variable templates as template parameters.</li>
<li class="core">A better <code>from_chars</code> interface.</li>
<li class="core">Template argument deduction for return types.</li>
<li class="library"><code>views::enumerate</code>.</li>
<li class="async">Threads should have a name.</li>
<li class="unicorn"><span>Replace <code>initializer_list</code> with language tuples.</span></li>
<li class="core">A strong <code>int8_t</code> type.</li>
<li class="text">Getting rid of <code>std::locale</code>.</li>
<li class="text">Getting rid of <code>char_traits</code>.</li>
<li class="core">Embedding resources in source files.</li>
<li class="perf">Container rellocation.</li>
<li class="library"><code>optional&lt;T&amp;&gt;</code>.</li>
<li class="perf">Uninitialized <code>vector</code>.</li>
<li class="core"><code>constexpr</code> warnings.</li>
<li class="core">More implicit <code>noexcept</code>.</li>
<li class="core">Structured bindings in <code>constexpr</code> contexts.</li>
<li class="library">More formatting support for library types.</li>
<li class="text"><code>u8/u16/u32</code> support in <code>std::format</code>.</li>
<li class="text"><code>char*</code> ⬄ <code>char8_t*</code> casting.</li>
<li class="library">Better hashing facilities.</li>
<li class="library">Correctly rounded floating points functions.</li>
<li class="core">Regular <code>void</code></li>
<li class="unicorn"><span>Fixing ADL</span>.</li>
<li class="unicorn"><span>Two phases parsing</span>.</li>
<li class="unicorn"><span>Breaking ABI.</span></li>
<li class="core">A pipeline operator for improved ranges and sender/receiver code usability.</li>
<li class="library">Views and iterators facades.</li>
<li class="library">null-terminated string views.</li>
<li class="unicorn"><span>Phasing out null-terminated string interfaces.</span></li>
<li class="library">Max buffer size for <code>to_chars</code>.</li>
<li class="library">Elastic integers.</li>
<li class="library">Safe overflow/saturating functions.</li>
<li class="library">Fixing <code>std::vector&lt;bool&gt;</code></li>
<li class="library">A replacement for <code>std::bitset</code>.</li>
<li class="library"><code>ranges::to</code> support for fixed-size arrays.</li>
<li class="library big">Improving the freestanding and embedded platforms story.</li>
<li class="library">Make more friends hidden to improve compile times and diagnostics.</li>
<li class="unicorn"><span>Improve the helpfulness of compilation errors.</span></li>
<li class="core">Nested structured bindings.</li>
<li class="core big">Pattern matching.</li>
<li class="core">Unused variable placeholder.</li>
<li class="text medium">Efficient, compile-time checked, Unicode compliant regexes.</li>
<li class="core">More ergonomic allocators.</li>
<li class="core">Better error management facilities.</li>
<li class="library">Language or library support for enum bit flags.</li>
<li class="library">An easier way to construct explicit types in range pipelines.</li>
<li class="library">Fast binary I/O separate from text facilities.</li>
<li class="core">Improve the usability of pointer to members.</li>
<li class="core">A set of replacement functions for <code>volatile</code>.</li>
<li class="core">Generalized <code>= delete</code>.</li>
<li class="core">Generalized <code>using</code>.</li>
<li class="unicorn"><span>ABI resilience features.</span></li>






































</ul>
