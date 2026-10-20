## Hi there 👋

Là, avec les éléments que tu as partagés, on comprend beaucoup mieux le problème. Tu n’as pas besoin de remplacer le FQDD du contrôleur : tu dois faire une correspondance entre le FQDD utilisé par idrac_system_info et les métadonnées du contrôleur récupérées directement via Redfish.

Dans ton exemple :

Champ	Valeur	Signification
Controller	AHCI.Slot.7-1	Identifiant technique (FQDD)
Name	BOSS-S1	Nom du contrôleur
Model	BOSS-S1	Modèle du contrôleur
PhysicalDisk.Model	SSDSCKKB240G8R	Modèle du disque Intel, pas celui du contrôleur

Le principe est donc de conserver AHCI.Slot.7-1 pour identifier le contrôleur, puis d’obtenir BOSS-S1 à partir de sa ressource Redfish. Il faut éviter de chercher Model directement dans le dictionnaire storage_status.Controller, puisque, d’après ta sortie, celui-ci contient une structure d’inventaire différente.

1. Playbook de correction

Je te propose d’abord de tester cette méthode en lecture seule. Elle interroge la collection Storage, puis chaque ressource référencée par @odata.id. C’est cohérent avec Redfish et avec les commandes d’inventaire documentées par Ansible.

Adapte les variables d’authentification à celles que tu utilises déjà.

- name: Lire la collection Storage Redfish
  ansible.builtin.uri:
    url: "https://{{ idrac_ip }}/redfish/v1/Systems/System.Embedded.1/Storage/"
    method: GET
    user: "{{ idrac_username }}"
    password: "{{ idrac_password }}"
    force_basic_auth: true
    validate_certs: false
    return_content: true
    status_code: 200
  register: storage_collection
  no_log: true
- name: Vérifier les membres renvoyés par Redfish
  ansible.builtin.debug:
    msg: "{{ storage_collection.json }}"
  when: storage_collection.json.Members is defined
- name: Lire chaque ressource Storage
  ansible.builtin.uri:
    url: "https://{{ idrac_ip }}{{ item['@odata.id'] }}"
    method: GET
    user: "{{ idrac_username }}"
    password: "{{ idrac_password }}"
    force_basic_auth: true
    validate_certs: false
    return_content: true
    status_code: 200
  loop: "{{ storage_collection.json.Members | default([]) }}"
  loop_control:
    label: "{{ item['@odata.id'] | default('URI absente') }}"
  register: storage_details
  no_log: true
- name: Afficher les noms et modèles des contrôleurs
  ansible.builtin.debug:
    msg:
      Id: "{{ item.json.Id | default('Non fourni') }}"
      Name: "{{ item.json.Name | default('Non fourni') }}"
      Model: >-
        {{ item.json.Model
           | default(
               (item.json.StorageControllers | default([{}]) | first).Model
               | default('Non fourni', true),
               true
             )
        }}
      Manufacturer: >-
        {{ item.json.Manufacturer
           | default(
               (item.json.StorageControllers | default([{}]) | first).Manufacturer
               | default('Non fourni', true),
               true
             )
        }}
  loop: "{{ storage_details.results | default([]) }}"
  when:
    - item.json is defined

2. Ce que tu dois vérifier dans le résultat

Pour le contrôleur de ta capture, tu veux obtenir quelque chose comme :

Id: AHCI.Slot.7-1
Name: BOSS-S1
Model: BOSS-S1
Manufacturer: Dell Inc.

C’est un résultat attendu d’après tes données, pas un résultat que j’ai pu tester sur ton iDRAC. Le modèle peut se trouver directement dans item.json.Model ou dans item.json.StorageControllers[0].Model, selon la structure réellement renvoyée.

Si le premier debug ne montre pas de Members, ne poursuis pas en supposant que la collection est correcte : regarde la réponse complète et le code HTTP. Certains firmwares peuvent exposer une structure différente.

3. Liens techniques vérifiés

* Ansible ⁠￼redfish_info⁠￼ — inventaire des contrôleurs de stockage⁠￼ : la commande GetStorageControllerInventory existe officiellement.
* Schéma officiel Redfish DMTF⁠￼ : référence des ressources et propriétés de stockage.
* Exemple d’inventaire Redfish sur GitHub⁠￼ : exemple de contrôleurs, disques et volumes dans les ressources Storage. C’est un projet tiers, pas une documentation officielle Dell.

Mon conseil : lance d’abord les trois premières tâches et regarde le JSON retourné pour AHCI.Slot.7-1. Si tu me transmets ce JSON (en masquant les informations sensibles), je pourrai adapter précisément la correspondance avec tes champs PhysicalDisk et VirtualDisk, sans inventer la structure de ton firmware.
