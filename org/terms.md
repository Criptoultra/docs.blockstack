---

description: "Blockstack Network documentation"

---
# Glossary

<table class="uk-table uk-table-large uk-table-striped">
{% for member in site.data.glossary %}
<tr>
    <th>{{ member.Term }}</th>
    <td>{{ member.Definition }}</td>
</tr>
{% endfor %}
</table>
