# La clé privée sur une YubiKey

> [!NOTE]
> Cette partie est **documentée mais pas montée** dans le lab. Pour mon lab la clé privée d'ADM01 est un simple fichier sur la machine. Ici j'explique comment on ferait avec une YubiKey.


## Pourquoi ?

Comme j'ai dit précédemment dans le [cours IPsec](01-cours-ipsec.md), IKEv2 accepte plusieurs méthodes d'authentification et parmi elles il y a l'authentification par **signature numérique avec certificat**. Et en fait la clé privée qui sert à signer soit on la stocke sur la machine, soit on la met dans un endroit plus sûr, par exemple un **jeton matériel** --> une clé de sécurité faite pour ça.

Dans le lab, la clé est dans `/etc/lab-pki/adm01.key`. Ce qui fait que quelqu'un qui a accès au disque (vol du PC, malware avec les droits root...) peut **copier la clé** et se faire passer pour ADM01 depuis n'importe où.

Avec un jeton matériel, la clé privée **ne sort jamais** du jeton. Les signatures sont calculées **à l'intérieur** --> la machine envoie juste les données à signer et récupère le résultat. Même root ne peut pas l'extraire.


## La YubiKey

On prend la **YubiKey** comme exemple. C'est une clé de sécurité matérielle qui permet de faire du MFA et elle supporte plusieurs protocoles comme FIDO2, OTP, OpenPGP, et **Smart Card (PIV)** --> et c'est pile ce qu'il nous faut dans notre cas.

**PIV** (Personal Identity Verification) c'est un standard de carte à puce. En gros, la YubiKey se comporte comme une carte à puce avec des **emplacements** (slots) qui contiennent chacun une paire de clés et un certificat :

| Slot | Usage prévu |
|:-----|:------------|
| **9a** | authentification --> c'est celui qu'on utilise pour IPsec |
| 9c | signature numérique (documents, mails) |
| 9d | gestion de clés (chiffrement) |
| 9e | authentification sans PIN (badge physique) |

> [!NOTE]
> On peut utiliser n'importe quel slot car ce sont surtout des conventions donc rien nous empêche d'utiliser le slot 9c par exemple (`handle = 02` dans strongSwan au lieu de `01`).
> Par contre chaque slot a sa politique de PIN par défaut : le 9c demande le PIN **à chaque signature**, et le 9e ne demande **jamais** de PIN, donc à éviter pour IPsec.

## Comment ça marche

En gros :
  1. On **génère la paire de clés dans la YubiKey** dans le slot 9a. La clé privée n'en sort plus jamais.
  2. On fait la **demande de certificat (CSR)** avec la clé publique. La CSR est signée par la YubiKey elle-même donc ça prouve à la CA qu'on possède bien la clé privée.
  3. On soumet la CSR à **LAB-Root-CA** exactement comme dans le lab (modèle IPsec-PosteAdmin, approbation manuelle).
  4. On récupère le certificat et on le **range dans la YubiKey** dans le même slot 9a.
  5. On configure **strongSwan** pour qu'il aille chercher la clé et le certificat dans la YubiKey au lieu d'un fichier.

Donc au lieu d'être stockée dans un fichier sur la machine, la clé privée est dans une clé matérielle protégée par un **code PIN** et on peut en plus exiger une **activation physique** (toucher la clé) à chaque utilisation.

> [!NOTE]
> Le certificat est **public**. Le mettre dans la YubiKey c'est surtout pratique (tout est au même endroit). Ce qui compte vraiment c'est que la **clé privée** y soit.

> [!TIP]
> Le PIN PIV fait **entre 6 et 8 caractères**. Après 3 erreurs --> le PIN est bloqué et il faut le PUK pour le débloquer. Le "toucher" n'est pas activé par défaut, c'est une option qu'on choisit au moment de générer la clé (`--touch-policy`).


## Les commandes

Avec `ykman` (YubiKey Manager) :

```bash
# 0. Changer le PIN, le PUK et la clé de gestion par défaut (à faire une seule fois)
ykman piv access change-pin
ykman piv access change-puk
ykman piv access change-management-key --generate --protect

# 1. Générer la paire de clés DANS la YubiKey (slot 9a)
#    PIN demandé à chaque signature + toucher obligatoire
ykman piv keys generate --algorithm ECCP384 \
      --pin-policy ALWAYS --touch-policy ALWAYS \
      9a adm01-pub.pem

# 2. Faire la CSR avec la clé de la YubiKey
ykman piv certificates request \
      --subject "CN=adm01-elyes,O=Lab,C=FR" \
      9a adm01-pub.pem adm01.csr

# 3. Sur CA01 : soumettre adm01.csr avec certreq (modèle IPsec-PosteAdmin),
#    approuver la demande puis récupérer adm01.cer (comme dans le lab)

# 4. Ranger le certificat dans le slot 9a
ykman piv certificates import 9a adm01.cer
```

> [!NOTE]
> On peut garder RSA si on veut (`--algorithm RSA2048`) mais ECCP384 est plus rapide sur la YubiKey et cohérent avec ECP-384 qu'on utilise déjà pour le DH.


## Côté strongSwan : le plugin pkcs11

Suite à ça, on configure strongSwan pour qu'il trouve la clé privée dans la YubiKey grâce au **plugin pkcs11**. **PKCS#11** c'est une  pour parler aux jetons matériels, strongSwan (le démon `charon`) demande "signe-moi ça avec la clé n°01" --> et c'est la YubiKey qui fait le calcul. Ce plugin permet à strongSwan d'utiliser la clé privée et le certificat stockés dans la clé.

Pour que ça marche, il faut aussi :
  - le service **pcscd** (il gère les lecteurs de cartes à puce) ;
  - une bibliothèque PKCS#11 pour la YubiKey : `libykcs11.so` (fournie avec `yubico-piv-tool`) ou `opensc-pkcs11.so` (OpenSC).

### 1. Déclarer le module dans `strongswan.conf`

```conf
charon {
  plugins {
    pkcs11 {
      modules {
        yubikey {
          path = /chemin/vers/libykcs11.so
        }
      }
    }
  }
}
```

### 2. La différence dans `swanctl.conf`

```diff
 connections {
   lab {
     version = 2
     remote_addrs = 10.10.50.1
     vips = 0.0.0.0
     proposals = aes256gcm16-prfsha384-ecp384

     local {
       auth = pubkey
-      # Version lab : le certificat est un fichier
-      certs = /etc/lab-pki/adm01.cer
+      # Version YubiKey : le certificat est lu dans le slot 9a
+      cert-yk {
+        module = yubikey
+        handle = 01        # 01 = slot 9a en PKCS#11
+      }
     }
     remote {
       auth = pubkey
       id = fw01.ad.lab.test
       cacerts = /etc/lab-pki/lab-root-ca.cer
     }
     children {
       admin {
         remote_ts = 10.10.10.0/24
         esp_proposals = aes256gcm16-ecp384
       }
     }
   }
 }

+secrets {
+  # Version YubiKey : la clé privée reste dans la YubiKey
+  token-yk {
+    module = yubikey
+    handle = 01
+    # pas de "pin = ..." : swanctl --load-creds demande le PIN au clavier
+  }
+}
```

Et dans la version lab, la clé était un fichier dans `/etc/swanctl/private/adm01.key`. Avec la YubiKey, **il n'y a plus aucun fichier de clé sur la machine**.

Le flux complet devient :

```bash
sudo swanctl --load-creds          # demande le PIN, charge le certificat depuis la YubiKey
sudo swanctl --initiate --child admin
# --> pendant IKE_AUTH, la YubiKey clignote : on la touche, elle signe le champ AUTH
```
