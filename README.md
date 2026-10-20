## Hi there 👋

<!--
**Tining-ramsses/Tining-ramsses** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: .g..
-->
J’ai trouvé ce qu’il te faut dans la documentation officielle d’Ansible, le schéma Redfish et un exemple publié par Dell. La clé est de récupérer la propriété Model du contrôleur de stockage, au lieu d’utiliser son Id.

Sur ta capture, on voit notamment :

* Controller: BOSS-S1 : le nom du contrôleur associé.
* Id: AHCI.Slot.7-1 : un identifiant technique, pas le modèle commercial.
* Manufacturer: INTEL et Model: SSDSC... : ces informations correspondent au SSD affiché, pas nécessairement au contrôleur.

Dell publie d’ailleurs un exemple Redfish réel dans lequel le contrôleur a Id: AHCI.Integrated.1-1, mais Model: BOSS-S1 et Name: BOSS-S1.

1. La solution dans ton playbook Ansible

Utilise la commande GetStorageControllerInventory du module community.general.redfish_info, conçue précisément pour récupérer l’inventaire des contrôleurs de stockage. Le résultat est accessible avec redfish_facts.storage_controller.entries.

- name: Récupérer les modèles des contrôleurs via Redfish
  community.general.redfish_info:
    category: Systems
    command: GetStorageControllerInventory
    baseuri: "https://{{ idrac_ip }}"
    username: "{{ idrac_username }}"
    password: "{{ idrac_password }}"
    validate_certs: false
  register: storage_controller_inventory
  no_log: true
- name: Afficher le nom et le modèle du contrôleur
  ansible.builtin.debug:
    msg:
      Controller: >-
        {{ item.Model
           | default(item.Name | default('Modèle inconnu', true), true) }}
  loop: "{{ storage_controller_inventory.redfish_facts.storage_controller.entries }}"
  loop_control:
    label: >-
      {{ item.Model
         | default(item.Name | default(item.Id, true), true) }}

Remplace idrac_ip, idrac_username et idrac_password par les variables de ton playbook. validate_certs: false est adapté uniquement si tu dois contourner le certificat non reconnu de l’iDRAC ; privilégie la validation du certificat lorsqu’elle est possible.

Si la collection n’est pas installée :

ansible-galaxy collection install community.general

2. Où vérifier que ça fonctionne réellement ?

Voici les liens directs que j’ai trouvés :

Documentation officielle Ansible

Description de redfish_info, commandes disponibles et exemples de récupération des propriétés.

Consulter la documentation Ansible⁠￼

Exemple concret fourni par Dell

L’inventaire d’un contrôleur BOSS-S1 comprend bien les champs Id, Model, Name et Manufacturer.

Voir l’exemple Redfish Dell sur GitHub⁠￼

Documentation officielle Dell sur les identifiants FQDD

Explique les identifiants techniques des composants et leurs noms plus lisibles.

Voir le guide Dell⁠￼

Schéma officiel Redfish DMTF

Définit les propriétés Model et Name de la ressource StorageController.

Consulter le schéma Redfish⁠￼

3. Attention à un détail important dans ton cas

Si ton playbook récupère actuellement les disques avec GetDiskInventory, ne remplace pas simplement le champ Id par Model dans cette même liste : tu risques d’afficher le modèle du SSD Intel au lieu de celui du contrôleur BOSS-S1.

Il faut récupérer l’inventaire des contrôleurs séparément, comme dans le playbook ci-dessus. Si tu veux ensuite afficher le modèle du contrôleur sur chaque ligne de disque, il faudra relier les entrées des deux inventaires à partir des références réellement renvoyées par ton iDRAC.

Envoie-moi le bloc de ton playbook qui récupère et affiche actuellement Controller, Id, Description, Manufacturer et Model. Je pourrai te montrer précisément quelle partie modifier sans chambouler le reste de ton automatisation.
