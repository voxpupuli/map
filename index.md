---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: map
---

<script src="https://unpkg.com/@joergdietrich/leaflet.terminator"></script>
<script language="javascript">
function linkTo(member, content) {
  return `<a href="${member.html_url}" target="_blank">${content}</a>`;
}
function renderMemberPopup(member) {
  return `
  <p>${linkTo(member, `<img src="${member.avatar_url}" width="72" class="avatar" />`)}</p>
  <p><strong class="login">${linkTo(member, member.login)}</strong> <span class="name">${linkTo(member, member.name)}</span></p>
  <p>${member.bio || ''}</p>
  `;
}
function memberPopup(layer) {
  member = layer.options.user_data;

  const el = document.createElement('div');
  el.className = 'member-popup';

  const popup = async (url) => {
    const response = await fetch(url);
    if (!response.ok) {
      alert(response.statusText);
    }

    const json = await response.json();
    el.innerHTML = renderMemberPopup(json);
  };

  popup('https://api.github.com/users/' + member.login);
  return el;
};
function callback(data) {
    var members = [];
    // When the map wrap, markers jump from one side of the map to the other.
    // To limit this, we have a copy of each marker which is beyond the map
    // limit and make these transitions smoother.
    var wrapped_members = [];

    var member_icon = L.icon({
        iconSize:      [25, 41],
        iconAnchor:    [12, 41],
        popupAnchor:   [1, -34],
        tooltipAnchor: [16, -28],
        shadowSize:    [41, 41],
        iconUrl:       '{{ "/marker.png" | relative_url }}',
        shadowUrl:     '{{ "/marker-shadow.png" | relative_url }}',
    });

    data.forEach(m => {
      members.push(L.marker([m.lat, m.lng], { icon: member_icon, title: m.login, user_data: m }).bindPopup(memberPopup, {minWidth: 200}));
      if (m.lng < 0) {
          wrapped_members.push(L.marker([m.lat, m.lng + 360], { icon: member_icon, title: m.login, user_data: m }).bindPopup(memberPopup, {minWidth: 200}));
      } else {
          wrapped_members.push(L.marker([m.lat, m.lng - 360], { icon: member_icon, title: m.login, user_data: m }).bindPopup(memberPopup, {minWidth: 200}));
      }
    });

    var common_attribution = 'Map data © <a href="https://www.openstreetmap.fr/">OpenStreetMap</a> | Tiles: ';
    var map_instructions = ' | <a href="{{ "/about.html" | relative_url }}">Manage your visibility on this map</a>';

    var neighbourhood = L.tileLayer('https://tile.thunderforest.com/neighbourhood/{z}/{x}/{y}.png?apikey={{ site.thunderforest_apikey }}', {attribution: common_attribution + 'Neighbourhood © <a href="https://thunderforest.com/">Thunderforest</a>' + map_instructions});

    var markers_layer = L.markerClusterGroup();

    var members_layer = L.featureGroup.subGroup(markers_layer, members.concat(wrapped_members));

    markers_layer.addLayer(members_layer);

    var map = L.map('map', {layers: [neighbourhood, markers_layer, members_layer], worldCopyJump: true});

    L.control.scale({maxWidth: 300}).addTo(map);

    var terminator = L.terminator({fillOpacity: 0.1, interactive: false});
    terminator.addTo(map);
    setInterval(function() { terminator.setTime(); }, 60000);

    new L.Control.BootstrapDropdowns({
        position: "topright",
        className: "menu",
        autoClose: "outside",
        menuItems: [
            {
                html: '<h5><i class="fa-solid fa-map-marked-alt"></i> Edit the pins on the map</h5><p>Use the GitHub actions below to add or remove yourself from the map. Use the parameters of the action to specify either an address or exact lat/long coordinates, to whatever degree of precision you are comfortable with.</p>',
                title: "Edit the pins on the map",
                header: true,
            },
            {
                separator: true,
            },
            {
                html: '<i class="fa-solid fa-user-plus"></i> Add yourself to the map',
                title: "Add yourself",
                href: "https://github.com/{{ site.github_project }}/actions/workflows/add_me_to_the_map.yml",
            },
            {
                html: '<i class="fa-solid fa-user-minus"></i> Remove yourself from the map',
                title: "Remove yourself",
                href: "https://github.com/{{ site.github_project }}/actions/workflows/remove_me_from_the_map.yml",
            },
            {
                separator: true,
            },
            {
                html: '<i class="fa-solid fa-info-circle"></i> About this map',
                title: "About",
                href: "/map/about.html",  // href without target
            }
        ],
    }).addTo(map);

    if (data.length >= {{ site.min_users_for_local_map }}) {
        map.fitBounds(L.latLngBounds(members.map(e => e.getLatLng())));
    } else {
        map.fitWorld();
    }
}

fetch('{{ "/data.json" | relative_url }}')
  .then(response => {
    if (!response.ok) {
      throw new Error(response.statusText);
    }
    return response.json();
  })
  .then(data => callback(data))
  .catch(error => alert('Error fetching data:', error));

</script>
